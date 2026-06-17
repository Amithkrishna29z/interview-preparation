# SQL Advanced Window Functions & Query Optimization
## Complete Interview Preparation Guide for Full Stack Java Developers

---

## 1. Window Functions — Complete Guide

### What Are Window Functions?

A **window function** performs a calculation across a set of rows that are related to the current row — called a "window" — without collapsing those rows into a single output row. This is the fundamental difference from GROUP BY aggregates.

**Key insight:** With GROUP BY, 10 rows can become 3 rows. With window functions, 10 rows always remain 10 rows, but each row gains an extra calculated column.

```sql
-- GROUP BY collapses rows — you lose individual row data
SELECT department_id, AVG(salary)
FROM employees
GROUP BY department_id;
-- Result: one row per department

-- Window function preserves rows — you keep all data + the calculation
SELECT
    employee_id,
    name,
    department_id,
    salary,
    AVG(salary) OVER (PARTITION BY department_id) AS dept_avg_salary
FROM employees;
-- Result: one row per employee, PLUS the department average alongside it
```

### Syntax Structure

```sql
function_name([arguments])
    OVER (
        [PARTITION BY partition_expression, ...]
        [ORDER BY sort_expression [ASC|DESC], ...]
        [frame_clause]
    )
```

Where `frame_clause` is:
```sql
{ ROWS | RANGE | GROUPS }
BETWEEN frame_start AND frame_end

-- frame_start / frame_end options:
UNBOUNDED PRECEDING   -- from the very first row of the partition
n PRECEDING           -- n rows before current row
CURRENT ROW           -- the current row itself
n FOLLOWING           -- n rows after current row
UNBOUNDED FOLLOWING   -- to the very last row of the partition
```

### Processing Order (Critical for Interviews)

SQL processes clauses in this order:

```
1. FROM / JOIN      — identify source tables
2. WHERE            — filter rows
3. GROUP BY         — group rows
4. HAVING           — filter groups
5. SELECT           — compute expressions (including window functions)
6. DISTINCT         — remove duplicates
7. ORDER BY         — sort final result
8. LIMIT / OFFSET   — pagination
```

**Window functions execute in step 5 (SELECT), AFTER WHERE, GROUP BY, and HAVING.**

This means:
- You CANNOT use a window function result in a WHERE clause directly
- You CANNOT use a window function result in a HAVING clause
- You CAN use window functions in ORDER BY (same level)
- To filter on window function results, wrap in a subquery or CTE

```sql
-- WRONG: cannot filter on window function in WHERE
SELECT employee_id, salary,
       RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rnk
FROM employees
WHERE rnk = 1;  -- ERROR: column "rnk" does not exist

-- CORRECT: wrap in subquery
SELECT * FROM (
    SELECT employee_id, salary,
           RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rnk
    FROM employees
) ranked
WHERE rnk = 1;
```

### Window Functions vs Aggregate Functions

| Feature | GROUP BY Aggregate | Window Function |
|---|---|---|
| Collapses rows | Yes | No |
| Can access individual row columns | No (only grouped/aggregated) | Yes |
| Result rows vs input rows | Fewer | Same count |
| Filter results with WHERE | No (use HAVING) | Use subquery/CTE |
| Access other rows in set | No | Yes (LAG, LEAD, etc.) |
| Partition data | Via GROUP BY only | PARTITION BY |

---

## 2. Ranking Functions

### Setup: Sample Data

```sql
-- Create and populate employees table for all ranking examples
CREATE TABLE employees (
    employee_id   INT PRIMARY KEY,
    name          VARCHAR(100),
    department    VARCHAR(50),
    salary        DECIMAL(10,2),
    hire_date     DATE,
    manager_id    INT
);

INSERT INTO employees VALUES
(1,  'Alice',   'Engineering', 95000,  '2019-01-15', NULL),
(2,  'Bob',     'Engineering', 85000,  '2020-03-10', 1),
(3,  'Charlie', 'Engineering', 85000,  '2020-06-01', 1),
(4,  'Diana',   'Engineering', 75000,  '2021-08-20', 1),
(5,  'Eve',     'Marketing',   70000,  '2019-05-01', NULL),
(6,  'Frank',   'Marketing',   65000,  '2020-09-15', 5),
(7,  'Grace',   'Marketing',   65000,  '2021-02-28', 5),
(8,  'Henry',   'Marketing',   60000,  '2022-01-10', 5),
(9,  'Iris',    'Finance',     90000,  '2018-07-01', NULL),
(10, 'Jack',    'Finance',     80000,  '2019-11-30', 9),
(11, 'Karen',   'Finance',     80000,  '2020-04-15', 9),
(12, 'Leo',     'Finance',     72000,  '2021-12-01', 9);
```

---

### ROW_NUMBER()

Assigns a **unique sequential integer** to each row within the partition. No ties — even if two rows have identical values, they get different numbers. The tiebreaker is arbitrary (or determined by a secondary ORDER BY).

```sql
SELECT
    employee_id,
    name,
    department,
    salary,
    ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS row_num
FROM employees;
```

**Result:**

| employee_id | name    | department  | salary | row_num |
|---|---|---|---|---|
| 1           | Alice   | Engineering | 95000  | 1       |
| 2           | Bob     | Engineering | 85000  | 2       |
| 3           | Charlie | Engineering | 85000  | 3       |
| 4           | Diana   | Engineering | 75000  | 4       |
| 9           | Iris    | Finance     | 90000  | 1       |
| 10          | Jack    | Finance     | 80000  | 2       |
| 11          | Karen   | Finance     | 80000  | 3       |
| 12          | Leo     | Finance     | 72000  | 4       |

Note: Bob and Charlie both earn 85000 but get row_num 2 and 3 (no tie handling).

**Use Case 1: Get the highest-paid employee per department**

```sql
SELECT department, name, salary
FROM (
    SELECT
        department, name, salary,
        ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS rn
    FROM employees
) t
WHERE rn = 1;
```

**Use Case 2: Deduplicate rows — keep one row per duplicate group**

```sql
-- Keep only the lowest order_id per (customer_id, order_date)
WITH deduped AS (
    SELECT order_id,
           ROW_NUMBER() OVER (
               PARTITION BY customer_id, order_date
               ORDER BY order_id
           ) AS rn
    FROM orders_raw
)
DELETE FROM orders_raw
WHERE order_id IN (SELECT order_id FROM deduped WHERE rn > 1);
```

**Use Case 3: Get the most recent order per customer**

```sql
CREATE TABLE orders (
    order_id INT PRIMARY KEY, customer_id INT, order_date DATE, amount DECIMAL(10,2)
);
INSERT INTO orders VALUES
(1, 101, '2024-01-10', 150.00), (2, 101, '2024-03-15', 200.00), (3, 101, '2024-05-20', 350.00),
(4, 102, '2024-02-01', 100.00), (5, 102, '2024-04-10', 250.00), (6, 103, '2024-06-01', 500.00);

-- Same rn = 1 pattern, partitioned by customer with most recent first
SELECT customer_id, order_id, order_date, amount
FROM (
    SELECT customer_id, order_id, order_date, amount,
        ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) AS rn
    FROM orders
) t
WHERE rn = 1;
```

---

### RANK()

Assigns the same rank to tied rows. After a tie, **gaps appear** in the ranking sequence (like real competition rankings: 1st, 1st, 3rd).

```sql
SELECT
    employee_id,
    name,
    department,
    salary,
    RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS rnk
FROM employees;
```

**Result for Engineering:**

| name    | salary | rnk |
|---|---|---|
| Alice   | 95000  | 1   |
| Bob     | 85000  | 2   |
| Charlie | 85000  | 2   |
| Diana   | 75000  | 4   |

Bob and Charlie both rank 2. Diana jumps to 4 (gap of 1 for the tie).

**Use Case: Competition-style rankings (sports, leaderboards)**

```sql
-- Game leaderboard: players with same score get same rank
CREATE TABLE game_scores (
    player_id   INT,
    player_name VARCHAR(50),
    score       INT,
    game_date   DATE
);

INSERT INTO game_scores VALUES
(1, 'PlayerA', 1500, '2024-01-01'),
(2, 'PlayerB', 1200, '2024-01-01'),
(3, 'PlayerC', 1500, '2024-01-01'),
(4, 'PlayerD', 900,  '2024-01-01'),
(5, 'PlayerE', 1200, '2024-01-01');

SELECT
    player_name,
    score,
    RANK() OVER (ORDER BY score DESC) AS competition_rank
FROM game_scores;
```

**Result:**

| player_name | score | competition_rank |
|---|---|---|
| PlayerA     | 1500  | 1                |
| PlayerC     | 1500  | 1                |
| PlayerB     | 1200  | 3                |
| PlayerE     | 1200  | 3                |
| PlayerD     | 900   | 5                |

---

### DENSE_RANK()

Same rank for ties, but **no gaps** in the ranking sequence (1, 1, 2, 3 — not 1, 1, 3).

```sql
SELECT
    employee_id,
    name,
    department,
    salary,
    DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dense_rnk
FROM employees;
```

**Result for Engineering:**

| name    | salary | dense_rnk |
|---|---|---|
| Alice   | 95000  | 1         |
| Bob     | 85000  | 2         |
| Charlie | 85000  | 2         |
| Diana   | 75000  | 3         |

Diana is rank 3 (not 4 like with RANK()).

**Use Case: Find employees in the top 3 salary tiers per department**

```sql
SELECT department, name, salary, dense_rnk
FROM (
    SELECT
        department, name, salary,
        DENSE_RANK() OVER (
            PARTITION BY department
            ORDER BY salary DESC
        ) AS dense_rnk
    FROM employees
) ranked
WHERE dense_rnk <= 3;
```

This returns all employees whose salary is in the top 3 distinct salary levels per department — even if multiple employees share a salary tier.

**Comparison: ROW_NUMBER vs RANK vs DENSE_RANK**

```sql
-- See all three side by side
SELECT
    name,
    department,
    salary,
    ROW_NUMBER()  OVER (PARTITION BY department ORDER BY salary DESC) AS row_num,
    RANK()        OVER (PARTITION BY department ORDER BY salary DESC) AS rnk,
    DENSE_RANK()  OVER (PARTITION BY department ORDER BY salary DESC) AS dense_rnk
FROM employees
WHERE department = 'Engineering';
```

| name    | salary | row_num | rnk | dense_rnk |
|---|---|---|---|---|
| Alice   | 95000  | 1       | 1   | 1         |
| Bob     | 85000  | 2       | 2   | 2         |
| Charlie | 85000  | 3       | 2   | 2         |
| Diana   | 75000  | 4       | 4   | 3         |

---

### NTILE(n)

Divides the rows within each partition into **n equal-sized buckets** (tiles) and assigns each row a bucket number from 1 to n. If rows don't divide evenly, earlier buckets get one extra row.

```sql
-- Divide Engineering employees into salary quartiles
SELECT
    name,
    department,
    salary,
    NTILE(4) OVER (PARTITION BY department ORDER BY salary DESC) AS quartile
FROM employees
WHERE department = 'Engineering';
```

**Result:**

| name    | salary | quartile |
|---|---|---|
| Alice   | 95000  | 1        |
| Bob     | 85000  | 2        |
| Charlie | 85000  | 3        |
| Diana   | 75000  | 4        |

**Use Case: Percentile grouping for A/B testing, commission tiers**

```sql
-- Assign customers to tiers via deciles. Use a CTE so the NTILE window
-- isn't repeated inside the CASE.
WITH customer_deciles AS (
    SELECT
        customer_id,
        total_spend,
        NTILE(10) OVER (ORDER BY total_spend DESC) AS decile
    FROM (
        SELECT customer_id, SUM(amount) AS total_spend
        FROM orders
        GROUP BY customer_id
    ) customer_totals
)
SELECT
    customer_id,
    total_spend,
    decile,
    CASE
        WHEN decile <= 2 THEN 'Platinum'
        WHEN decile <= 5 THEN 'Gold'
        ELSE 'Standard'
    END AS tier
FROM customer_deciles;
```

---

## 3. Offset Functions

Offset functions let you access values from **other rows** relative to the current row within the window — without a self-join.

### Setup: Daily Sales Data

```sql
CREATE TABLE daily_sales (
    sale_date DATE PRIMARY KEY,
    revenue   DECIMAL(12,2),
    region    VARCHAR(50)
);

INSERT INTO daily_sales VALUES
('2024-01-01', 10000, 'North'),
('2024-01-02', 12000, 'North'),
('2024-01-03', 9500,  'North'),
('2024-01-04', 15000, 'North'),
('2024-01-05', 13000, 'North'),
('2024-01-06', 11000, 'North'),
('2024-01-07', 14000, 'North'),
('2024-01-08', 16000, 'North');
```

---

### LAG(column, offset, default)

Returns the value of `column` from the row that is `offset` rows **before** the current row within the partition. If no such row exists, returns `default` (or NULL if omitted).

```sql
-- Calculate day-over-day revenue change
SELECT
    sale_date,
    revenue,
    LAG(revenue, 1, 0) OVER (ORDER BY sale_date) AS prev_day_revenue,
    revenue - LAG(revenue, 1, 0) OVER (ORDER BY sale_date) AS daily_change,
    ROUND(
        (revenue - LAG(revenue) OVER (ORDER BY sale_date))
        / LAG(revenue) OVER (ORDER BY sale_date) * 100,
        2
    ) AS pct_change
FROM daily_sales
ORDER BY sale_date;
```

**Result:**

| sale_date  | revenue | prev_day | daily_change | pct_change |
|---|---|---|---|---|
| 2024-01-01 | 10000   | 0        | 10000        | NULL       |
| 2024-01-02 | 12000   | 10000    | 2000         | 20.00      |
| 2024-01-03 | 9500    | 12000    | -2500        | -20.83     |
| 2024-01-04 | 15000   | 9500     | 5500         | 57.89      |

**Use Case: Find gaps in sequences**

```sql
CREATE TABLE user_logins (
    user_id    INT,
    login_date DATE
);

INSERT INTO user_logins VALUES
(1, '2024-01-01'), (1, '2024-01-02'), (1, '2024-01-04'),
(1, '2024-01-05'), (1, '2024-01-08'), (1, '2024-01-09');

-- Find dates where a user was absent (gap > 1 day).
-- Remember: window results can't go in WHERE directly, so wrap in a subquery.
SELECT user_id, prev_login, curr_login, gap_days
FROM (
    SELECT
        user_id,
        login_date AS curr_login,
        LAG(login_date) OVER (PARTITION BY user_id ORDER BY login_date) AS prev_login,
        login_date - LAG(login_date) OVER (PARTITION BY user_id ORDER BY login_date) AS gap_days
    FROM user_logins
) t
WHERE gap_days > 1;
```

A year-over-year comparison is the same idea with a larger offset: `LAG(monthly_rev, 12) OVER (ORDER BY month)` compares each month to the same month last year.

---

### LEAD(column, offset, default)

Returns the value of `column` from the row `offset` rows **after** the current row. The mirror of LAG.

```sql
-- Calculate time to next event (useful for session analysis)
CREATE TABLE page_views (
    session_id  INT,
    page_name   VARCHAR(100),
    view_time   TIMESTAMP
);

INSERT INTO page_views VALUES
(1, 'Home',     '2024-01-01 10:00:00'),
(1, 'Products', '2024-01-01 10:02:30'),
(1, 'Cart',     '2024-01-01 10:05:00'),
(1, 'Checkout', '2024-01-01 10:07:45'),
(2, 'Home',     '2024-01-01 11:00:00'),
(2, 'Blog',     '2024-01-01 11:05:00');

SELECT
    session_id,
    page_name,
    view_time,
    LEAD(view_time) OVER (PARTITION BY session_id ORDER BY view_time) AS next_page_time,
    LEAD(page_name) OVER (PARTITION BY session_id ORDER BY view_time) AS next_page,
    EXTRACT(EPOCH FROM
        LEAD(view_time) OVER (PARTITION BY session_id ORDER BY view_time) - view_time
    ) AS seconds_on_page
FROM page_views;
```

LEAD is also handy for churn analysis: compute `LEAD(order_date)` per customer to measure the gap to each customer's next order, then flag customers whose most recent order has no follow-up within N days.

---

### FIRST_VALUE(column) and LAST_VALUE(column)

Returns the first or last value of `column` in the window frame.

**Critical note about LAST_VALUE:** By default, the window frame is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`. This means LAST_VALUE returns the current row's value (not the last row of the partition). You must explicitly set the frame to get the true last value.

```sql
-- FIRST_VALUE: straightforward
SELECT
    name,
    department,
    salary,
    FIRST_VALUE(name) OVER (
        PARTITION BY department ORDER BY salary DESC
    ) AS highest_earner_in_dept
FROM employees;

-- LAST_VALUE: requires explicit frame extension
SELECT
    name,
    department,
    salary,
    -- WRONG: returns current row's name (default frame ends at current row)
    LAST_VALUE(name) OVER (
        PARTITION BY department ORDER BY salary DESC
    ) AS wrong_lowest_earner,

    -- CORRECT: extend frame to end of partition
    LAST_VALUE(name) OVER (
        PARTITION BY department
        ORDER BY salary DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS correct_lowest_earner
FROM employees;
```

With the full-partition frame, you can attach the department min/max salary to every row — `FIRST_VALUE(salary)` for the max (ORDER BY DESC) and `LAST_VALUE(salary)` for the min.

---

### NTH_VALUE(column, n)

Returns the value of `column` from the nth row in the window frame (remember to extend the frame to the full partition).

```sql
-- Get the 2nd highest salary in each department
SELECT DISTINCT
    department,
    NTH_VALUE(salary, 2) OVER (
        PARTITION BY department
        ORDER BY salary DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS second_highest_salary
FROM employees;
```

---

## 4. Aggregate Window Functions

Aggregate functions (SUM, AVG, COUNT, MIN, MAX) can all be used as window functions by adding an OVER clause.

### Running Total (Cumulative Sum)

```sql
SELECT
    sale_date,
    revenue,
    SUM(revenue) OVER (ORDER BY sale_date) AS running_total,
    SUM(revenue) OVER (ORDER BY sale_date
                       ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
                      ) AS running_total_explicit  -- same as above
FROM daily_sales;
```

**Result:**

| sale_date  | revenue | running_total |
|---|---|---|
| 2024-01-01 | 10000   | 10000         |
| 2024-01-02 | 12000   | 22000         |
| 2024-01-03 | 9500    | 31500         |
| 2024-01-04 | 15000   | 46500         |

### 7-Day Moving Average

```sql
SELECT
    sale_date,
    revenue,
    ROUND(
        AVG(revenue) OVER (
            ORDER BY sale_date
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        ),
        2
    ) AS moving_avg_7d
FROM daily_sales;
```

The frame `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW` means: current row + 6 rows before = 7 rows total.

### Percent of Total

```sql
SELECT
    sale_date,
    revenue,
    SUM(revenue) OVER () AS grand_total,
    ROUND(revenue / SUM(revenue) OVER () * 100, 2) AS pct_of_total,
    ROUND(revenue / SUM(revenue) OVER (
        ORDER BY sale_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) * 100, 2) AS running_pct_of_cumulative
FROM daily_sales;
```

Note: `SUM(revenue) OVER ()` — empty OVER clause means the entire result set is one partition, giving the grand total.

### Other Aggregate Windows (same patterns)

`COUNT`, `MIN`, and `MAX` work as running aggregates the same way — just swap the function in the `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` frame. A fixed moving sum uses a bounded frame, e.g. `SUM(revenue) OVER (ORDER BY sale_date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)` for a 3-day rolling sum. You can also mix scopes in one query: `SUM(revenue) OVER (PARTITION BY region)` for a per-region total alongside `SUM(revenue) OVER ()` for the grand total, then divide to get percentages.

---

## 5. PARTITION BY Clause

PARTITION BY divides the rows of the query result into **independent groups** (partitions). The window function is applied separately within each partition. It resets for each partition.

### PARTITION BY vs GROUP BY

```sql
-- GROUP BY: one row per department — individual employee data is gone
SELECT department, AVG(salary) AS avg_salary
FROM employees
GROUP BY department;

-- PARTITION BY: one row per employee — keeps all columns
SELECT
    employee_id, name, department, salary,
    AVG(salary) OVER (PARTITION BY department) AS dept_avg_salary,
    salary - AVG(salary) OVER (PARTITION BY department) AS diff_from_avg
FROM employees;
```

**PARTITION BY** is to window functions what **GROUP BY** is to aggregates — but PARTITION BY never removes rows.

### Multiple PARTITION BY Columns

```sql
CREATE TABLE regional_sales (
    region      VARCHAR(50),
    product     VARCHAR(50),
    year        INT,
    quarter     INT,
    revenue     DECIMAL(12,2)
);

-- Rank products within each region-year combination
SELECT
    region,
    product,
    year,
    revenue,
    RANK() OVER (
        PARTITION BY region, year
        ORDER BY revenue DESC
    ) AS rank_in_region_year
FROM regional_sales;
```

### No PARTITION BY = Entire Result Set Is One Window

```sql
-- Rank all employees by salary across the entire company (no partition)
SELECT
    name,
    department,
    salary,
    RANK() OVER (ORDER BY salary DESC) AS company_rank
FROM employees;
```

---

## 6. ORDER BY in Window Functions

ORDER BY inside the OVER clause determines the order of rows **within each partition** for the calculation.

### Without ORDER BY

When you omit ORDER BY inside OVER, the entire partition is the frame for every row. This makes sense for aggregate window functions (every row sees the full partition aggregate):

```sql
-- No ORDER BY: each row gets the SAME value (full partition aggregate)
SELECT
    name,
    department,
    salary,
    SUM(salary) OVER (PARTITION BY department) AS total_dept_salary,
    AVG(salary) OVER (PARTITION BY department) AS avg_dept_salary
FROM employees;
```

### With ORDER BY

When you include ORDER BY inside OVER, the **default frame becomes** `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`. This means each row only sees rows up to and including itself (great for running totals):

```sql
-- With ORDER BY: running total (each row sees an accumulation)
SELECT
    sale_date,
    revenue,
    SUM(revenue) OVER (ORDER BY sale_date) AS running_total
FROM daily_sales;
```

### ORDER BY Direction

```sql
-- Ascending salary rank within department
ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary ASC)

-- Descending (highest paid = rank 1)
ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC)

-- Multiple sort keys
ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC, hire_date ASC)
```

---

## 7. Frame Specification: ROWS vs RANGE

The frame clause defines the subset of rows within the partition that the window function considers for the current row.

### ROWS: Physical Row Count

ROWS counts actual rows — it is exact and predictable. The common frames:

```sql
-- Running total (start to current)
SUM(revenue) OVER (ORDER BY sale_date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)

-- Trailing 3-row window (2 preceding + current)
SUM(revenue) OVER (ORDER BY sale_date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)

-- 3-row centered window (1 before, current, 1 after)
AVG(revenue) OVER (ORDER BY sale_date ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING)

-- Entire partition (explicit)
SUM(revenue) OVER (ORDER BY sale_date ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)
```

### RANGE: Logical Range Based on Values

RANGE operates on the **values** of the ORDER BY column, not physical row positions. Rows with equal ORDER BY values are treated as "peers" and included together.

```sql
-- ROWS treats each row separately; RANGE groups equal-valued rows
SELECT
    score,
    SUM(score) OVER (ORDER BY score ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)  AS rows_cumsum,
    SUM(score) OVER (ORDER BY score RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS range_cumsum
FROM tied_example;  -- rows with score=10,10,20,30
```

For two rows with score=10, both `range_cumsum` values are 20 (peers summed together), while `rows_cumsum` gives 10 then 20. Prefer ROWS for predictability; reach for RANGE only when equal values should be one group.

### Awareness: GROUPS and Named Windows

- **GROUPS** (PostgreSQL 11+) is like ROWS but counts by groups of equal values, e.g. `GROUPS BETWEEN 2 PRECEDING AND CURRENT ROW`.
- **Named windows** let you define a window once and reuse it: add `WINDOW w AS (PARTITION BY department ORDER BY salary DESC)` after the FROM clause and write `RANK() OVER w`, `LAG(salary) OVER w`, etc. Cleaner when the same window repeats.

---

## 8. Complex Window Function Examples

### Top N Per Group

The most common interview pattern: rank inside a subquery/CTE, then filter.

```sql
-- Top 2 highest-paid employees per department
SELECT department, name, salary
FROM (
    SELECT
        department, name, salary,
        DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS rnk
    FROM employees
) ranked
WHERE rnk <= 2;
```

Use `DENSE_RANK` to include ties, or `ROW_NUMBER() OVER (... ORDER BY salary DESC, employee_id)` for exactly N rows with a deterministic tiebreak.

### Awareness: Advanced Patterns

These come up in senior interviews; know the idea, not the exact syntax:

- **Islands and gaps**: number consecutive rows with `ROW_NUMBER()`, then group by `date - rn` (the constant "island key") to collapse consecutive runs into ranges.
- **Centered vs trailing moving average**: `ROWS BETWEEN 3 PRECEDING AND 3 FOLLOWING` (centered) vs `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW` (trailing 7-day).
- **Year-over-year**: aggregate to monthly, then compare with `LAG(revenue, 12)` (guard the denominator with `NULLIF`).
- **Median / quartiles**: `PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary)` per group.
- **Mode**: group + count, then keep the top `RANK() OVER (ORDER BY COUNT(*) DESC)`.

---

## 9. Common Table Expressions (CTEs)

### Basic CTE Syntax

```sql
WITH cte_name AS (
    SELECT ...
    FROM ...
    WHERE ...
),
second_cte AS (
    SELECT ...
    FROM cte_name
    WHERE ...
)
SELECT *
FROM second_cte;
```

### CTE vs Subquery vs Temp Table

| Aspect | Subquery | CTE | Temp Table |
|---|---|---|---|
| Readability | Poor for complex queries | Excellent | Good |
| Reusability in same query | No (must repeat) | Yes (reference by name) | Yes |
| Materialized | No (usually) | Depends on DB | Yes |
| Indexes | No | No | Yes (can add) |
| Scope | Local to statement | Local to statement | Session-wide |
| Recursive | No | Yes | No |
| Use when | Simple, one-time | Complex multi-step, recursive | Large datasets, reused across queries |

```sql
-- Same logic: subquery vs CTE

-- Subquery (hard to read):
SELECT name, salary
FROM (
    SELECT name, salary, AVG(salary) OVER (PARTITION BY department) AS avg_sal
    FROM employees
) t
WHERE salary > avg_sal;

-- CTE (readable):
WITH dept_avg AS (
    SELECT name, salary, AVG(salary) OVER (PARTITION BY department) AS avg_sal
    FROM employees
)
SELECT name, salary
FROM dept_avg
WHERE salary > avg_sal;
```

### Multiple CTEs

```sql
-- Chain multiple CTEs to build complex analysis step by step
WITH
-- Step 1: Calculate monthly revenue
monthly_revenue AS (
    SELECT
        DATE_TRUNC('month', order_date) AS month,
        customer_id,
        SUM(amount) AS monthly_spend
    FROM orders
    GROUP BY 1, 2
),

-- Step 2: Add monthly rank per customer
ranked_months AS (
    SELECT
        month,
        customer_id,
        monthly_spend,
        RANK() OVER (PARTITION BY customer_id ORDER BY monthly_spend DESC) AS spend_rank
    FROM monthly_revenue
),

-- Step 3: Get customers' best month
best_months AS (
    SELECT customer_id, month AS best_month, monthly_spend AS peak_spend
    FROM ranked_months
    WHERE spend_rank = 1
)

-- Final: join back to customers
SELECT
    c.customer_id,
    c.name,
    bm.best_month,
    bm.peak_spend
FROM customers c
JOIN best_months bm USING (customer_id)
ORDER BY peak_spend DESC;
```

---

## 10. Recursive CTEs

Recursive CTEs solve hierarchical and graph problems that are impossible (or very hard) with regular SQL.

### Syntax

```sql
WITH RECURSIVE cte_name AS (
    -- Anchor member: base case (non-recursive)
    SELECT ...
    FROM ...
    WHERE ...

    UNION ALL

    -- Recursive member: references cte_name
    SELECT ...
    FROM ... JOIN cte_name ON ...
    WHERE ...  -- termination condition (must eventually become false)
)
SELECT * FROM cte_name;
```

### Use Case 1: Employee Hierarchy (Org Chart)

```sql
-- Employees table already has manager_id (self-referential)
-- Traverse from top-level managers down to all their reports

WITH RECURSIVE org_chart AS (
    -- Anchor: start with top-level employees (no manager)
    SELECT
        employee_id,
        name,
        manager_id,
        0 AS depth,
        name::TEXT AS path
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- Recursive: join employees with their manager found in previous iteration
    SELECT
        e.employee_id,
        e.name,
        e.manager_id,
        oc.depth + 1,
        oc.path || ' -> ' || e.name
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.employee_id
)
SELECT
    employee_id,
    name,
    depth,
    path
FROM org_chart
ORDER BY path;
```

**Result:**

| employee_id | name    | depth | path                        |
|---|---|---|---|
| 1           | Alice   | 0     | Alice                       |
| 2           | Bob     | 1     | Alice -> Bob                |
| 3           | Charlie | 1     | Alice -> Charlie            |
| 4           | Diana   | 2     | Alice -> Bob -> Diana       |

### Awareness: Other Recursive Patterns

The same anchor + recursive-member structure solves many problems:

- **Category tree** (e-commerce): walk `parent_id` from a root category down to all descendants, carrying a `level` and `path` for display.
- **Generate a sequence**: `SELECT 1 AS n UNION ALL SELECT n + 1 FROM nums WHERE n < 100` produces numbers 1–100; the same trick with `d + INTERVAL '1 day'` generates date ranges.
- **Bill of materials**: explode a multi-level component hierarchy, multiplying quantities down each level to get total cost.

**Infinite loop prevention:** Always ensure the recursive member has a WHERE clause that eventually returns no rows. PostgreSQL also caps depth via `max_recursion_depth` (default 100).

---

## 11. Advanced Query Techniques

### LATERAL Joins

A LATERAL join lets a subquery on the right side reference columns from tables on the left (like a correlated subquery, but in the FROM clause). Useful for top-N per group and for table functions that take parameters from the outer query.

```sql
-- Top 2 orders per customer (often faster than window functions on large tables)
SELECT c.customer_id, c.name, o.order_id, o.amount
FROM customers c
CROSS JOIN LATERAL (
    SELECT order_id, amount
    FROM orders
    WHERE customer_id = c.customer_id
    ORDER BY amount DESC
    LIMIT 2
) o;
```

It also pairs with set-returning functions, e.g. `FROM customers, LATERAL UNNEST(tags_array) AS tag` to expand an array column into rows.

---

### Self Joins

A self join joins a table to itself — for hierarchies or comparing rows within the same table.

```sql
-- Employees who earn MORE than their direct manager
SELECT
    e.name AS employee, e.salary AS emp_salary,
    m.name AS manager,  m.salary AS mgr_salary
FROM employees e
JOIN employees m ON e.manager_id = m.employee_id
WHERE e.salary > m.salary;
```

To compare pairs within a group (e.g. employees in the same department), join the table to itself on the group column and add `e1.employee_id < e2.employee_id` to avoid duplicate and self-pairs.

### Anti-Join Pattern

Find rows in table A that have NO match in table B. Two safe, generally-equivalent approaches:

```sql
-- NOT EXISTS (handles NULLs correctly, optimizer can use an index)
SELECT c.customer_id, c.name
FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id
);

-- LEFT JOIN ... IS NULL (same performance in most optimizers)
SELECT c.customer_id, c.name
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE o.customer_id IS NULL;
```

Avoid `NOT IN` here: if the subquery returns a single NULL, the whole query returns no rows. If you must use it, add `WHERE customer_id IS NOT NULL` to the subquery.

### Non-Equi Joins

```sql
-- Find all salary grade assignments (salary falls in a range)
CREATE TABLE salary_grades (
    grade      CHAR(1),
    min_salary DECIMAL(10,2),
    max_salary DECIMAL(10,2)
);

INSERT INTO salary_grades VALUES
('A', 50000, 69999),
('B', 70000, 89999),
('C', 90000, 999999);

SELECT e.name, e.salary, sg.grade
FROM employees e
JOIN salary_grades sg
    ON e.salary BETWEEN sg.min_salary AND sg.max_salary;
```

---

## 12. EXPLAIN and Query Optimization

### Reading EXPLAIN ANALYZE (PostgreSQL)

Run `EXPLAIN ANALYZE <query>` to see the real execution plan. The numbers per node read as `(cost=startup..total rows=estimated width=bytes)` plus `actual time=... rows=... loops=...`. What to watch for as a junior:

- **Seq Scan on a large table** → likely a missing index.
- **estimated rows far from actual rows** → stale statistics (run `ANALYZE`).
- **high-cost nodes** → focus optimization there first.

(Awareness only: scan types are Seq Scan, Index Scan, Index-Only Scan, and Bitmap Heap Scan; join types are Nested Loop, Hash Join, and Merge Join — the planner picks based on table size, selectivity, and available indexes.)

### Indexing Strategy

The default **B-tree** index covers `=`, `<`, `>`, `BETWEEN`, `ORDER BY`, and `LIKE 'prefix%'`:

```sql
CREATE INDEX idx_employees_salary ON employees(salary);
CREATE INDEX idx_employees_dept_salary ON employees(department, salary);  -- composite
```

Other index types you should recognize:

- **Partial** — only rows matching a condition: `CREATE INDEX ... ON orders(customer_id) WHERE status = 'ACTIVE'` (smaller, faster).
- **Expression** — index a transformed column: `CREATE INDEX ... ON employees(LOWER(email))` (the query must use the same expression).
- **Covering** — `CREATE INDEX ... ON employees(department) INCLUDE (name, salary)` enables index-only scans.
- **GIN** — for arrays, full-text, and JSONB containment queries.

### Left-Prefix Rule for Composite Indexes

```sql
CREATE INDEX idx_composite ON employees(department, salary, hire_date);

-- USES index (left-prefix matched):
WHERE department = 'Engineering'
WHERE department = 'Engineering' AND salary > 80000
WHERE department = 'Engineering' AND salary > 80000 AND hire_date > '2020-01-01'

-- DOES NOT fully use index (skips left column):
WHERE salary > 80000                    -- no department filter
WHERE hire_date > '2020-01-01'          -- no department/salary filter
WHERE department = 'Engineering' AND hire_date > '2020-01-01'  -- skips salary
```

### Query Optimization Techniques

**1. Avoid functions on indexed columns in WHERE:**
```sql
-- BAD: function on indexed column prevents index use
WHERE YEAR(hire_date) = 2020
WHERE LOWER(email) = 'alice@example.com'  -- unless expression index exists
WHERE salary / 12 > 7000

-- GOOD: transform the constant instead
WHERE hire_date BETWEEN '2020-01-01' AND '2020-12-31'
WHERE email = 'alice@example.com'  -- store emails lowercase consistently
WHERE salary > 84000
```

**2. Keyset pagination vs OFFSET:**
```sql
-- BAD: OFFSET pagination — DB scans and discards N rows every time
SELECT * FROM orders ORDER BY order_id LIMIT 20 OFFSET 10000;
-- On page 500 of 20 items: scans 10,000 rows to discard them

-- GOOD: keyset pagination — O(log n) with index
SELECT * FROM orders
WHERE order_id > :last_seen_id  -- pass the last order_id from previous page
ORDER BY order_id
LIMIT 20;
```

**3. Avoid SELECT *:**
```sql
-- BAD: transfers all columns, may prevent index-only scans
SELECT * FROM employees WHERE department = 'Engineering';

-- GOOD: only what you need
SELECT employee_id, name, salary FROM employees WHERE department = 'Engineering';
```

**4. Correlated subqueries — rewrite with JOINs:**
```sql
-- BAD: correlated subquery runs once per row
SELECT name, salary
FROM employees e
WHERE salary > (
    SELECT AVG(salary) FROM employees WHERE department = e.department
);
-- Runs AVG subquery for every row in employees

-- GOOD: window function or JOIN
SELECT name, salary
FROM (
    SELECT name, salary,
           AVG(salary) OVER (PARTITION BY department) AS dept_avg
    FROM employees
) t
WHERE salary > dept_avg;
-- Calculates department averages once
```

**5. N+1 query problem (critical for Java developers):**
```sql
-- Java code that causes N+1:
-- for each customer: SELECT * FROM orders WHERE customer_id = ?
-- This runs 1 query for customers + N queries for their orders

-- Solution: JOIN in one query
SELECT c.customer_id, c.name, o.order_id, o.amount
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id;

-- Or use EXISTS to check without fetching all order data:
SELECT c.customer_id, c.name
FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id);
```

**6. ANALYZE and VACUUM (PostgreSQL):** Run `ANALYZE employees` to refresh planner statistics after big data loads, and `VACUUM ANALYZE employees` to also reclaim space from dead tuples. Autovacuum usually handles this in the background.

---

## 13. Advanced Aggregation

### GROUPING SETS

Execute multiple GROUP BY specifications in a single query:

```sql
-- Equivalent to UNION ALL of multiple GROUP BY queries
SELECT department, EXTRACT(YEAR FROM hire_date) AS year, COUNT(*) AS headcount
FROM employees
GROUP BY GROUPING SETS (
    (department, EXTRACT(YEAR FROM hire_date)),  -- by dept AND year
    (department),                                 -- by dept only
    (EXTRACT(YEAR FROM hire_date)),               -- by year only
    ()                                            -- grand total
);
```

### ROLLUP

Generates subtotals and a grand total (hierarchical aggregation):

```sql
-- region+department totals, region subtotals (department=NULL), and grand total
SELECT region, department, SUM(revenue) AS total_revenue
FROM regional_sales
GROUP BY ROLLUP(region, department);
```

(The `GROUPING(col)` function returns 1 for subtotal rows, useful to replace the NULLs with a label like `'ALL REGIONS'`.)

### CUBE

Generates all possible combinations of GROUP BY columns:

```sql
-- CUBE(a, b, c) = GROUPING SETS((a,b,c),(a,b),(a,c),(b,c),(a),(b),(c),())
SELECT region, product, year, SUM(revenue)
FROM regional_sales
GROUP BY CUBE(region, product, year);
```

### FILTER Clause on Aggregates

Apply a WHERE condition to specific aggregate functions:

```sql
SELECT
    department,
    COUNT(*) AS total_employees,
    COUNT(*) FILTER (WHERE salary > 80000) AS high_earners,
    AVG(salary) FILTER (WHERE hire_date >= '2020-01-01') AS avg_salary_new_hires,
    SUM(salary) FILTER (WHERE manager_id IS NULL) AS total_exec_salary
FROM employees
GROUP BY department;
```

### PERCENTILE_CONT / PERCENTILE_DISC

`PERCENTILE_CONT` interpolates between values (continuous); `PERCENTILE_DISC` returns an actual value (discrete). Both use the `WITHIN GROUP (ORDER BY ...)` syntax:

```sql
SELECT
    department,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary) AS median_salary,
    PERCENTILE_CONT(0.9) WITHIN GROUP (ORDER BY salary) AS p90_salary
FROM employees
GROUP BY department;
```

(Awareness: **hypothetical-set aggregates** like `RANK(85000) WITHIN GROUP (ORDER BY salary DESC)` answer "what rank would this value get?" — rarely needed at junior level.)

---

## 14. JSON in SQL

### PostgreSQL JSONB

```sql
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    name       VARCHAR(100),
    metadata   JSONB
);

INSERT INTO products VALUES
(1, 'Laptop', '{"brand": "Dell", "specs": {"ram": 16, "storage": 512}, "tags": ["laptop", "business"]}'),
(2, 'Phone',  '{"brand": "Apple", "specs": {"ram": 8, "storage": 256}, "tags": ["phone", "premium"]}');

-- -> returns JSONB, ->> returns text
SELECT
    name,
    metadata -> 'brand'              AS brand_jsonb,    -- {"Dell"}
    metadata ->> 'brand'             AS brand_text,     -- Dell
    metadata -> 'specs' ->> 'ram'    AS ram             -- 16
FROM products;

-- @> containment operator: does the left JSONB contain the right?
SELECT * FROM products
WHERE metadata @> '{"brand": "Dell"}';

-- GIN index for fast JSONB queries
CREATE INDEX idx_products_metadata ON products USING GIN(metadata);
```

Awareness: other handy JSONB tools include `jsonb_array_elements_text(...)` to expand an array into rows, `jsonb_each_text(...)` to expand key/value pairs, and `jsonb_set(...)` to update a nested value.

### MySQL JSON

MySQL uses functions instead of operators: `JSON_EXTRACT(metadata, '$.brand')`, the `->>'$.brand'` shorthand (unquoted text), and `JSON_ARRAYAGG` / `JSON_OBJECTAGG` for aggregation. To index a JSON value, add a generated column and index that:

```sql
ALTER TABLE products ADD COLUMN brand VARCHAR(100)
    GENERATED ALWAYS AS (metadata->>'$.brand') STORED;
CREATE INDEX idx_brand ON products(brand);
```

---

## 15. Transactions and Locking

### Isolation Levels

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read | Performance |
|---|---|---|---|---|
| READ UNCOMMITTED | Possible | Possible | Possible | Fastest |
| READ COMMITTED | No | Possible | Possible | Fast (default in most DBs) |
| REPEATABLE READ | No | No | Possible | Medium (default in MySQL InnoDB) |
| SERIALIZABLE | No | No | No | Slowest |

```sql
-- Set isolation level for a transaction
BEGIN;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
SELECT * FROM accounts WHERE account_id = 1;
UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;
COMMIT;
```

**Dirty Read:** Read uncommitted data from another transaction that might roll back.
**Non-Repeatable Read:** Read same row twice in one transaction, get different values (another transaction committed an UPDATE between reads).
**Phantom Read:** Execute same query twice, get different rows (another transaction committed an INSERT/DELETE between reads).

### Pessimistic Locking

```sql
-- SELECT FOR UPDATE: lock rows so no other transaction can modify them
BEGIN;
SELECT * FROM inventory
WHERE product_id = 101
FOR UPDATE;  -- locks the row

UPDATE inventory
SET quantity = quantity - 1
WHERE product_id = 101;
COMMIT;

-- SKIP LOCKED: queue pattern — skip rows locked by other transactions
-- Perfect for job queues (multiple workers processing jobs concurrently)
BEGIN;
SELECT * FROM job_queue
WHERE status = 'PENDING'
ORDER BY created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;  -- skip rows locked by other workers
-- Process the job...
UPDATE job_queue SET status = 'PROCESSING' WHERE job_id = :id;
COMMIT;

-- SELECT FOR SHARE: allow concurrent reads but prevent writes
SELECT * FROM products WHERE product_id = 1 FOR SHARE;
```

### Optimistic Locking

No database lock — use a version column to detect conflicts:

```sql
-- Schema: add version column
ALTER TABLE accounts ADD COLUMN version INT DEFAULT 0;

-- Java pattern: read version, update only if version unchanged
-- Step 1: Read
SELECT account_id, balance, version FROM accounts WHERE account_id = 1;
-- Returns: balance=1000, version=5

-- Step 2: Update (include version in WHERE)
UPDATE accounts
SET balance = balance - 100, version = version + 1
WHERE account_id = 1 AND version = 5;  -- fails if version changed concurrently

-- Check affected rows: if 0, another transaction modified the row (conflict!)
-- Java: if (rowsAffected == 0) throw new OptimisticLockException();
```

### Deadlock Detection and Prevention

A deadlock happens when transaction 1 holds row A and wants row B, while transaction 2 holds row B and wants row A. The main prevention rule: **always acquire locks in the same order** (e.g. ascending ID). Databases also auto-detect deadlocks and kill one transaction (PostgreSQL `deadlock_timeout`, default 1s); your app should catch the error and retry.

(Awareness: PostgreSQL **advisory locks** — `pg_advisory_lock(key)` / `pg_try_advisory_lock(key)` — are application-level locks not tied to a row, handy for ensuring a single instance of a distributed cron job runs.)

---

## 16. Classic SQL Interview Problems

### Problem 1: Second Highest Salary Per Department

```sql
-- Setup
CREATE TABLE emp_salaries (
    emp_id INT, name VARCHAR(50), dept VARCHAR(50), salary DECIMAL(10,2)
);
INSERT INTO emp_salaries VALUES
(1,'Alice','Eng',95000),(2,'Bob','Eng',85000),(3,'Charlie','Eng',85000),
(4,'Diana','Eng',75000),(5,'Eve','Mkt',70000),(6,'Frank','Mkt',65000);

-- Window function (most elegant) — DENSE_RANK so ties on top salary still
-- yield the right "second" tier; filter dr = 2 in a subquery or CTE.
SELECT dept, name, salary
FROM (
    SELECT dept, name, salary,
           DENSE_RANK() OVER (PARTITION BY dept ORDER BY salary DESC) AS dr
    FROM emp_salaries
) t
WHERE dr = 2;
```

(Without window functions, a correlated subquery works: keep rows where exactly one distinct salary in the same dept is higher — `WHERE 1 = (SELECT COUNT(DISTINCT salary) FROM emp_salaries e2 WHERE e2.dept = e1.dept AND e2.salary > e1.salary)`.)

### Problem 2: Employees Who Earn More Than Their Manager

```sql
SELECT e.name AS employee, e.salary AS emp_salary,
       m.name AS manager, m.salary AS mgr_salary
FROM employees e
JOIN employees m ON e.manager_id = m.employee_id
WHERE e.salary > m.salary;
```

### Problem 3: Find and Delete Duplicates (Keep One)

```sql
CREATE TABLE contacts (
    id    INT,
    email VARCHAR(100),
    name  VARCHAR(100)
);
INSERT INTO contacts VALUES
(1,'a@x.com','Alice'),(2,'b@x.com','Bob'),(3,'a@x.com','Alice Dup'),(4,'c@x.com','Carol');

-- Find duplicates
SELECT email, COUNT(*) AS cnt
FROM contacts
GROUP BY email
HAVING COUNT(*) > 1;

-- Delete duplicates, keep lowest id
DELETE FROM contacts
WHERE id NOT IN (
    SELECT MIN(id)
    FROM contacts
    GROUP BY email
);

-- PostgreSQL alternative using CTE
WITH dupes AS (
    SELECT id,
           ROW_NUMBER() OVER (PARTITION BY email ORDER BY id) AS rn
    FROM contacts
)
DELETE FROM contacts
WHERE id IN (SELECT id FROM dupes WHERE rn > 1);
```

### Problem 4: Running Total and Moving Average

```sql
SELECT
    sale_date,
    revenue,
    SUM(revenue) OVER (ORDER BY sale_date
                       ROWS UNBOUNDED PRECEDING)   AS running_total,
    ROUND(AVG(revenue) OVER (ORDER BY sale_date
                              ROWS BETWEEN 6 PRECEDING AND CURRENT ROW), 2) AS moving_avg_7d
FROM daily_sales;
```

### Problem 5: Find Gaps in a Sequence

```sql
CREATE TABLE sequence_data (id INT PRIMARY KEY);
INSERT INTO sequence_data VALUES (1),(2),(3),(5),(6),(9),(10),(11);

-- Method 1: self-join / correlated subquery
SELECT id + 1 AS gap_start
FROM sequence_data s1
WHERE NOT EXISTS (SELECT 1 FROM sequence_data s2 WHERE s2.id = s1.id + 1)
  AND id < (SELECT MAX(id) FROM sequence_data);

-- Method 2: LAG
SELECT curr_id - prev_id - 1 AS gap_size,
       prev_id + 1           AS gap_start,
       curr_id - 1           AS gap_end
FROM (
    SELECT id AS curr_id,
           LAG(id) OVER (ORDER BY id) AS prev_id
    FROM sequence_data
) t
WHERE curr_id - prev_id > 1;
```

### Problem 6: Cohort Retention Analysis

(Senior-level; awareness only.) Build a `cohorts` CTE mapping each user to their signup month, a `user_activity` CTE of distinct (user, active month) pairs, then `LEFT JOIN` them, compute `months_since_signup`, and divide `COUNT(DISTINCT active users)` by `COUNT(DISTINCT cohort users)` for the retention percentage.

### Problem 7: Pivot Table

The portable way to pivot rows to columns is conditional aggregation:

```sql
-- (dept, year, revenue) -> dept | 2022 | 2023 | 2024
SELECT
    department,
    SUM(CASE WHEN year = 2022 THEN revenue ELSE 0 END) AS "2022",
    SUM(CASE WHEN year = 2023 THEN revenue ELSE 0 END) AS "2023",
    SUM(CASE WHEN year = 2024 THEN revenue ELSE 0 END) AS "2024"
FROM dept_revenue
GROUP BY department;
```

(PostgreSQL also has the `crosstab` function in the `tablefunc` extension; to unpivot columns back to rows, use `CROSS JOIN LATERAL (VALUES ...)`.)

### Problem 8: Customers Who Bought A But Not B

```sql
-- Using NOT EXISTS (most readable)
SELECT DISTINCT o.customer_id
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
WHERE oi.product_id = 'A'
  AND NOT EXISTS (
      SELECT 1
      FROM orders o2
      JOIN order_items oi2 ON o2.order_id = oi2.order_id
      WHERE o2.customer_id = o.customer_id
        AND oi2.product_id = 'B'
  );

-- Using EXCEPT (set operation):
SELECT customer_id FROM orders JOIN order_items USING (order_id) WHERE product_id = 'A'
EXCEPT
SELECT customer_id FROM orders JOIN order_items USING (order_id) WHERE product_id = 'B';
```

### Problem 9: Top 3 Products Per Category by Sales

```sql
SELECT category, product_name, total_sales
FROM (
    SELECT
        p.category,
        p.product_name,
        SUM(oi.quantity * oi.unit_price) AS total_sales,
        RANK() OVER (
            PARTITION BY p.category
            ORDER BY SUM(oi.quantity * oi.unit_price) DESC
        ) AS sales_rank
    FROM products p
    JOIN order_items oi ON p.product_id = oi.product_id
    GROUP BY p.category, p.product_name
) ranked
WHERE sales_rank <= 3
ORDER BY category, sales_rank;
```

### Problem 10: Month-over-Month Growth Rate

```sql
WITH monthly AS (
    SELECT
        DATE_TRUNC('month', order_date)::DATE AS month,
        SUM(amount) AS revenue
    FROM orders
    GROUP BY 1
)
SELECT
    month,
    revenue,
    LAG(revenue) OVER (ORDER BY month) AS prev_month_revenue,
    ROUND(
        (revenue - LAG(revenue) OVER (ORDER BY month))
        / NULLIF(LAG(revenue) OVER (ORDER BY month), 0) * 100,
        2
    ) AS mom_growth_pct
FROM monthly
ORDER BY month;
```

---

## 17. Interview Questions & Answers (40+)

### Window Function Fundamentals

**Q1: What is the difference between ROW_NUMBER(), RANK(), and DENSE_RANK()?**

A: All three assign a number to rows within a partition ordered by some column, but they differ in how they handle ties:
- **ROW_NUMBER()**: Always assigns unique sequential numbers. Ties get different numbers (1, 2, 3, 4).
- **RANK()**: Tied rows get the same number. Numbers after a tie skip (1, 1, 3, 4) — like a real competition where two 1st-place finishers skip the 2nd-place rank.
- **DENSE_RANK()**: Tied rows get the same number. Numbers never skip (1, 1, 2, 3).

Use ROW_NUMBER when you need exactly one row per group (deduplication, latest record per entity). Use RANK/DENSE_RANK when ties have meaning.

---

**Q2: What is the difference between ROWS and RANGE in window frame specifications?**

A: Both define which rows are included in the window frame, but differently:
- **ROWS**: counts physical rows. `ROWS BETWEEN 2 PRECEDING AND CURRENT ROW` always includes exactly 3 rows.
- **RANGE**: uses the logical value of the ORDER BY column. `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` includes all rows with ORDER BY values less than or equal to the current row's value. If 3 rows have the same salary, they are all included in each other's RANGE frame — but ROWS would treat them as separate physical rows.

ROWS is more predictable; RANGE is useful when equal values should be treated as a group.

---

**Q3: Why doesn't LAST_VALUE() work as expected by default?**

A: Because the default window frame is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`. This frame ends at the current row, so LAST_VALUE() returns the current row's value. To get the actual last value of the partition, you must override the frame:
```sql
LAST_VALUE(col) OVER (
    PARTITION BY x ORDER BY y
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
)
```

---

**Q4: Can you use a window function in a WHERE clause?**

A: No. Window functions are evaluated in the SELECT phase, after WHERE and HAVING. To filter based on a window function result, you must wrap the query in a subquery or CTE:
```sql
-- Correct approach:
SELECT * FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY dept ORDER BY salary DESC) AS rn
    FROM employees
) t
WHERE rn = 1;
```

---

**Q5: What does PARTITION BY do? How is it different from GROUP BY?**

A: PARTITION BY divides rows into groups for the window function but does NOT reduce the number of rows in the result. Each row still appears in the output, but each window function calculation is scoped to the row's partition. GROUP BY reduces rows — each group becomes one row, and you can only reference grouped/aggregated columns.

---

### CTEs and Recursive Queries

**Q6: When would you use a CTE instead of a subquery?**

A: CTEs are preferable when:
1. The same result needs to be referenced multiple times in the query (subquery would require duplication)
2. The query has multiple logical steps — CTEs make each step named and readable
3. You need recursion (CTEs can be recursive; subqueries cannot)
4. You want to use window functions and then filter on their results (CTE gives clean separation)

Subqueries are fine for simple, one-off filtering.

---

**Q7: What is a recursive CTE? Give a real-world example.**

A: A recursive CTE is a CTE that references itself. It has two parts joined by UNION ALL: an anchor member (base case, non-recursive) and a recursive member (references the CTE). The recursion continues until the recursive member returns no rows.

Real-world example: traversing an employee org chart where each employee has a manager_id pointing to another employee. A recursive CTE can start from the CEO and traverse down to all reports at any depth — something impossible with regular JOINs when the depth is unknown.

---

**Q8: How do you prevent infinite recursion in a recursive CTE?**

A: By ensuring the recursive member's WHERE clause eventually becomes false. Common techniques:
1. Track depth and add `WHERE depth < 100`
2. Track visited node IDs in an array and check `NOT (node_id = ANY(visited))`
3. PostgreSQL enforces a default maximum of 100 iterations (`max_recursion_depth`)

---

### Query Optimization

**Q9: How do you use EXPLAIN ANALYZE to optimize a query?**

A: Run `EXPLAIN ANALYZE <query>` to see the actual execution plan. Key things to look for:
1. **Sequential Scan on a large table** — indicates a missing index
2. **Large discrepancy between estimated rows and actual rows** — means table statistics are stale (run ANALYZE)
3. **High cost nodes** — focus optimization effort on the most expensive operations
4. **Nested Loop with many loops** — may indicate the inner side lacks an index
5. **Sort nodes** — can be eliminated with the right index

---

**Q10: What is the difference between an index scan and a sequential scan? When does the optimizer choose each?**

A: A **sequential scan** reads every row in the table from disk. A **index scan** uses a B-tree to jump directly to matching rows. The optimizer chooses based on estimated selectivity:
- If the query returns a small fraction of rows (<5-15%), an index scan is usually faster
- If the query returns most of the table, a sequential scan may be faster (sequential disk I/O is faster than random I/O from index lookups)
- For very small tables, sequential scan is always chosen (overhead of index scan isn't worth it)

---

**Q11: When should you NOT add an index?**

A:
1. **Write-heavy tables**: every INSERT, UPDATE, DELETE must also update all indexes — too many indexes degrade write performance
2. **Small tables**: a full scan of a tiny table is faster than an index lookup
3. **Low-cardinality columns**: an index on a boolean or gender column with few distinct values is largely useless (scanning 50% of a table via index has more overhead than a sequential scan)
4. **Columns never used in WHERE, JOIN, or ORDER BY**
5. **Columns frequently updated**: index maintenance overhead is high

---

**Q12: What is a covering index?**

A: A covering index includes all the columns needed by a query, enabling an **index-only scan** — the database can answer the query entirely from the index without accessing the actual table (heap). This is significantly faster because it avoids random I/O.

```sql
-- Query needs name and salary filtered by department
SELECT name, salary FROM employees WHERE department = 'Engineering';

-- Covering index: department in the key, name and salary in INCLUDE
CREATE INDEX idx_emp_covering ON employees(department) INCLUDE (name, salary);
-- Now the query can be answered from the index alone
```

---

**Q13: How do you handle pagination efficiently at scale?**

A: Avoid OFFSET-based pagination for large offsets. OFFSET = N forces the database to scan and discard N rows.

Use **keyset pagination** (also called cursor-based or seek method):
```sql
-- Instead of: LIMIT 20 OFFSET 10000
-- Use: remember the last seen value and filter from there
SELECT * FROM orders
WHERE order_id > :last_seen_id
ORDER BY order_id
LIMIT 20;
```

This is O(log n) with an index vs O(n) for OFFSET. Trade-off: keyset pagination doesn't support jumping to arbitrary pages.

---

**Q14: Explain the N+1 query problem and how SQL solves it.**

A: N+1 occurs when code runs 1 query to fetch a list of N items, then runs a separate query for each item to fetch related data — resulting in N+1 total queries. This is devastating for performance.

SQL solution: JOIN everything in one query, or use `IN` / `EXISTS`:
```sql
-- N+1 problem in Java/Hibernate: fetches each order's items separately
-- Solution: JOIN or use Hibernate's @BatchSize / fetch join
SELECT c.*, o.*
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE c.customer_id IN (1, 2, 3);
```

---

**Q15: What is the difference between WHERE and HAVING?**

A: WHERE filters rows **before** grouping; HAVING filters groups **after** GROUP BY. You cannot use aggregate functions in WHERE; you must use HAVING for that. Window functions can appear in neither WHERE nor HAVING (they execute after both).

```sql
SELECT department, AVG(salary)
FROM employees
WHERE hire_date > '2020-01-01'  -- filters individual rows before grouping
GROUP BY department
HAVING AVG(salary) > 75000;    -- filters groups after aggregation
```

---

### Advanced Aggregation

**Q16: What is ROLLUP? Give an example.**

A: ROLLUP is a GROUP BY extension that generates subtotals and a grand total. `GROUP BY ROLLUP(a, b)` produces groupings `(a, b)`, `(a)`, and `()` (grand total). It's hierarchical — subtotals are computed for each level of the hierarchy from right to left.

```sql
SELECT region, product, SUM(sales)
FROM sales_data
GROUP BY ROLLUP(region, product);
-- Row 1-N: region + product combinations
-- Then: regional subtotals (product = NULL)
-- Then: grand total (region = NULL, product = NULL)
```

---

**Q17: What is GROUPING SETS?**

A: GROUPING SETS allows you to specify multiple GROUP BY clauses in a single query, equivalent to UNION ALL of multiple GROUP BY queries but much more efficient. `GROUP BY GROUPING SETS((a,b),(a),(b),())` computes all four groupings in one pass.

---

### Joins and Set Operations

**Q18: What are all the join types and when do you use each?**

A:
- **INNER JOIN**: returns rows that have matches in both tables. Most common.
- **LEFT JOIN**: all rows from left table + matching rows from right (NULLs for non-matches). Use when you want all left rows even without a match (e.g., all customers, including those with no orders).
- **RIGHT JOIN**: reverse of LEFT JOIN. Rarely needed (just swap table order and use LEFT JOIN).
- **FULL OUTER JOIN**: all rows from both tables, NULLs where no match. Use for finding rows in either table that have no match in the other.
- **CROSS JOIN**: Cartesian product — every row from left paired with every row from right. Use for generating combinations (e.g., size × color permutations for products).
- **SELF JOIN**: join a table to itself. Use for hierarchical data or row comparisons.

---

**Q19: What is the performance difference between NOT EXISTS and NOT IN?**

A: NOT EXISTS is generally safer and can be faster:
1. **NULL handling**: `NOT IN` returns no rows if the subquery contains a single NULL value (because `x NOT IN (1, NULL)` evaluates as UNKNOWN, not TRUE/FALSE). NOT EXISTS handles NULLs correctly.
2. **Performance**: With an index on the correlated column, NOT EXISTS uses an efficient index lookup. NOT IN may cause the optimizer to evaluate the subquery as a list. Modern optimizers often transform both to the same plan, but NOT EXISTS is the safer default.

---

**Q20: What is a FULL OUTER JOIN and when do you use it?**

A: FULL OUTER JOIN returns all rows from both tables. If a row in either table has no match, the columns from the other table are NULL. Use cases:
- Reconciling two data sources to find mismatches
- Finding rows in either table that have no corresponding row in the other

```sql
-- Find mismatches between two tables
SELECT a.id AS table_a_id, b.id AS table_b_id
FROM table_a a
FULL OUTER JOIN table_b b ON a.id = b.id
WHERE a.id IS NULL OR b.id IS NULL;
```

---

### Locking and Transactions

**Q21: What are the four transaction isolation levels and what anomalies does each prevent?**

A:
- **READ UNCOMMITTED**: No protection. Allows dirty reads, non-repeatable reads, phantom reads. Rarely used.
- **READ COMMITTED**: Prevents dirty reads. Each statement sees a fresh snapshot. Default in PostgreSQL and SQL Server.
- **REPEATABLE READ**: Prevents dirty and non-repeatable reads. Once you read a row, it won't change within the transaction. Default in MySQL InnoDB.
- **SERIALIZABLE**: Prevents all anomalies. Transactions behave as if they executed serially. Highest isolation, lowest concurrency.

---

**Q22: What is SKIP LOCKED and when do you use it?**

A: SKIP LOCKED is used with SELECT FOR UPDATE to skip rows that are already locked by another transaction. The classic use case is a **job queue**: multiple workers can process jobs concurrently without blocking each other — each worker picks the next available (unlocked) job.

```sql
-- Worker 1 and Worker 2 both run this; they get different jobs
SELECT job_id FROM jobs
WHERE status = 'PENDING'
ORDER BY created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;
```

---

**Q23: Explain optimistic vs pessimistic locking.**

A:
- **Pessimistic locking**: Assume conflicts will happen. Lock the row when you read it (`SELECT FOR UPDATE`). No one else can modify it until you commit. Safe but reduces concurrency.
- **Optimistic locking**: Assume conflicts are rare. Read the row with a version number. When updating, check the version hasn't changed. If it has, another transaction modified the row — retry or throw an error. Higher concurrency, but requires retry logic.

Pessimistic is better for high-contention scenarios (financial transactions). Optimistic is better for low-contention scenarios (user profile updates).

---

### JSON and Advanced Features

**Q24: What is the difference between -> and ->> in PostgreSQL JSONB?**

A: Both extract from JSONB, but with different return types:
- `->` returns the value as **JSONB** (preserves JSON type)
- `->>` returns the value as **text** (always a string)

```sql
metadata -> 'specs'          -- returns: {"ram": 16, "storage": 512}  (JSONB)
metadata ->> 'brand'         -- returns: Dell  (text, no quotes)
metadata -> 'specs' -> 'ram' -- returns: 16  (JSONB number)
metadata -> 'specs' ->> 'ram'-- returns: 16  (text "16")
```

Use `->` when you need to chain operators or pass to JSON functions. Use `->>` when you want a plain text/numeric value for comparison or display.

---

**Q25: How do you index a JSONB column for fast queries?**

A:
```sql
-- GIN index: supports @> (containment), ?, key existence
CREATE INDEX idx_metadata ON products USING GIN(metadata);

-- jsonb_path_ops variant: smaller, faster for @> only
CREATE INDEX idx_metadata_path ON products USING GIN(metadata jsonb_path_ops);

-- For specific path queries (generated column approach in PostgreSQL/MySQL):
ALTER TABLE products
ADD COLUMN brand TEXT GENERATED ALWAYS AS (metadata->>'brand') STORED;
CREATE INDEX idx_brand ON products(brand);
```

---

### Miscellaneous

**Q26: How do you find the Nth highest salary without window functions?**

A: Using a correlated subquery — count how many distinct salaries are higher:
```sql
-- Nth highest salary
SELECT DISTINCT salary
FROM employees e1
WHERE N-1 = (  -- replace N with the desired rank number
    SELECT COUNT(DISTINCT salary)
    FROM employees e2
    WHERE e2.salary > e1.salary
);
-- For 2nd highest: WHERE 1 = (COUNT ...)
```

But window functions are cleaner:
```sql
SELECT salary FROM (
    SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS dr
    FROM employees
) t WHERE dr = N;
```

---

**Q27: What is the difference between UNION and UNION ALL?**

A: Both combine result sets from two queries, but:
- **UNION**: eliminates duplicate rows (performs a DISTINCT operation) — slower
- **UNION ALL**: keeps all rows including duplicates — faster

Use UNION ALL by default unless you specifically need deduplication.

---

**Q28: How does the COALESCE function work and when do you use it?**

A: COALESCE returns the first non-NULL value from its argument list. Use it to handle NULL values in calculations, provide defaults, and clean up NULL display:

```sql
SELECT
    name,
    COALESCE(phone, email, 'No contact info') AS contact,
    COALESCE(bonus, 0) + salary AS total_compensation,
    -- Prevent division by zero:
    revenue / NULLIF(expense, 0) AS ratio
FROM employees;
```

---

**Q29: What is a lateral join vs a regular join?**

A: In a regular JOIN, the right side is evaluated independently of the left side. In a LATERAL join, the right side (subquery) can reference columns from the left side — like a correlated subquery but in the FROM clause. This enables top-N per group patterns and applying table functions that take parameters from the outer query.

---

**Q30: Explain GROUPING SETS, ROLLUP, and CUBE — when would you use each?**

A:
- **GROUPING SETS**: full control — specify exactly which combinations you need. Use when you need specific non-hierarchical groupings.
- **ROLLUP(a, b, c)**: hierarchical subtotals — all combinations going left to right, plus grand total. Use for reports with subtotals (e.g., Year > Quarter > Month).
- **CUBE(a, b, c)**: all 2^n combinations. Use for OLAP-style data exploration where you need every possible aggregation combination.

---

**Q31: What is index bloat and how do you fix it?**

A: Index bloat is dead entries left in an index by deleted/updated rows, wasting space and slowing queries. PostgreSQL autovacuum normally handles it; you can also rebuild manually with `REINDEX INDEX idx_name` or `REINDEX TABLE employees`.

---

**Q32: What is a partial index and when should you use one?**

A: A partial index only includes rows matching a WHERE condition. Use when:
- Only a small subset of rows is frequently queried (e.g., only active records)
- You want to enforce uniqueness only for a subset of rows

```sql
-- Index only active orders (smaller, faster than full index)
CREATE INDEX idx_active_orders_date ON orders(order_date)
WHERE status = 'ACTIVE';

-- Unique constraint only on non-deleted records
CREATE UNIQUE INDEX idx_unique_active_email ON users(email)
WHERE deleted_at IS NULL;
```

---

**Q33: How do you calculate a cumulative distribution (percent rank)?**

A:
```sql
SELECT
    name,
    salary,
    PERCENT_RANK() OVER (ORDER BY salary) AS percent_rank,
    -- PERCENT_RANK = (rank - 1) / (total_rows - 1)
    -- 0 = lowest, 1 = highest

    CUME_DIST() OVER (ORDER BY salary) AS cumulative_dist
    -- CUME_DIST = (number of rows <= current) / total_rows
    -- Always > 0, can equal 1
FROM employees;
```

---

**Q34: What is a deadlock and how do you prevent it in SQL?**

A: A deadlock occurs when two transactions each hold a lock that the other needs — they wait for each other forever. Prevention strategies:
1. **Consistent lock ordering**: always acquire locks in the same order across transactions
2. **Short transactions**: minimize the time locks are held
3. **NOWAIT or lock timeout**: fail fast instead of waiting
4. **Application-level retry logic**: catch deadlock exceptions and retry
5. **PostgreSQL advisory locks**: coordinate application-level locking before DB locks

---

**Q35: What is the difference between a B-tree and a Hash index?**

A:
- **B-tree**: supports equality (`=`), range (`<`, `>`, `BETWEEN`), sorting, and LIKE with a prefix. This is the default index type, suitable for most use cases.
- **Hash index**: only supports equality (`=`). Smaller and faster for equality lookups, but useless for range queries, sorting, or LIKE. PostgreSQL hash indexes are crash-safe since PostgreSQL 10.

---

**Q36: How do window functions affect performance compared to GROUP BY?**

A: Window functions generally add overhead because they must sort data within partitions and maintain a sliding frame. However, they eliminate the need for self-joins or correlated subqueries — which are far worse. Best practices:
1. Create indexes on PARTITION BY and ORDER BY columns used in window functions
2. Use the WINDOW clause to define a window once when reusing it multiple times
3. Filter rows before window functions execute (use WHERE/CTE)
4. Prefer ROWS over RANGE when possible (more predictable, often faster)

---

**Q37: What is the FILTER clause and how does it differ from CASE WHEN inside an aggregate?**

A: Both achieve conditional aggregation, but FILTER is cleaner and can be slightly more efficient:
```sql
-- Equivalent, but FILTER is preferred style:
COUNT(*) FILTER (WHERE status = 'ACTIVE')
COUNT(CASE WHEN status = 'ACTIVE' THEN 1 END)

-- FILTER also works with window functions:
SUM(amount) FILTER (WHERE category = 'Electronics') OVER (PARTITION BY region)
```

---

**Q38: How would you find all employees at every level of a reporting hierarchy under a specific manager?**

A: Use a recursive CTE:
```sql
WITH RECURSIVE reports AS (
    SELECT employee_id, name, manager_id, 0 AS level
    FROM employees
    WHERE employee_id = :target_manager_id  -- start here

    UNION ALL

    SELECT e.employee_id, e.name, e.manager_id, r.level + 1
    FROM employees e
    JOIN reports r ON e.manager_id = r.employee_id
)
SELECT * FROM reports ORDER BY level, name;
```

---

**Q39: What does NULLIF do?**

A: `NULLIF(a, b)` returns NULL if `a = b`, otherwise returns `a`. Most commonly used to prevent division-by-zero errors:
```sql
SELECT revenue / NULLIF(cost, 0) AS margin  -- returns NULL instead of error when cost=0
```

---

**Q40: How do you write a query that returns the top 3 products per category without using window functions?**

A: Using a correlated subquery (illustrates why window functions were invented):
```sql
SELECT category, product_name, total_sales
FROM product_sales ps1
WHERE (
    SELECT COUNT(*)
    FROM product_sales ps2
    WHERE ps2.category = ps1.category
      AND ps2.total_sales > ps1.total_sales
) < 3  -- fewer than 3 products have higher sales in this category
ORDER BY category, total_sales DESC;
```

This is far less efficient than the window function approach (runs a subquery per row). Use DENSE_RANK() OVER (PARTITION BY category ORDER BY total_sales DESC) in practice.

---

**Q41: What is the difference between DISTINCT and GROUP BY?**

A: They often produce the same result for simple cases, but:
- **DISTINCT**: removes duplicate rows from SELECT output. Simple deduplication.
- **GROUP BY**: groups rows for aggregation. Required when you use aggregate functions (COUNT, SUM, etc.).

```sql
-- These return the same result when no aggregation needed:
SELECT DISTINCT department FROM employees;
SELECT department FROM employees GROUP BY department;

-- But only GROUP BY supports aggregates:
SELECT department, COUNT(*), AVG(salary) FROM employees GROUP BY department;
```

DISTINCT is simpler; GROUP BY is necessary for any aggregation.

---

**Q42: How do you pivot data in SQL without a PIVOT keyword?**

A: Use conditional aggregation with CASE WHEN:
```sql
SELECT
    product,
    SUM(CASE WHEN quarter = 'Q1' THEN revenue END) AS Q1,
    SUM(CASE WHEN quarter = 'Q2' THEN revenue END) AS Q2,
    SUM(CASE WHEN quarter = 'Q3' THEN revenue END) AS Q3,
    SUM(CASE WHEN quarter = 'Q4' THEN revenue END) AS Q4
FROM quarterly_sales
GROUP BY product;
```

PostgreSQL also has the `crosstab` function in the tablefunc extension for dynamic pivots.

---

**Q43: What is a materialized CTE vs a regular CTE?**

A: A **materialized CTE** is computed once and stored; a non-materialized one is "inlined" into the query like a subquery so the optimizer can push predicates through it. PostgreSQL 12+ lets you force either with `WITH x AS MATERIALIZED (...)` or `AS NOT MATERIALIZED (...)`. Use MATERIALIZED when the CTE is expensive and referenced multiple times.

---

**Q44: How would you detect and fix slow queries in a production PostgreSQL database?**

A: Find the worst offenders via the `pg_stat_statements` extension (sort by `mean_exec_time`), run `EXPLAIN ANALYZE` on them, and look for sequential scans on large tables, stale row estimates, expensive sorts, and nested loops with many iterations. Fix by adding missing/covering indexes, running `ANALYZE`, and rewriting correlated subqueries. Check `pg_stat_activity` for locking, and set `log_min_duration_statement` to log slow queries going forward.

---

*End of SQL Advanced Window Functions & Query Optimization Guide*

---

## Quick Reference Summary

### Window Function Syntax
```sql
fn() OVER (PARTITION BY col ORDER BY col ROWS BETWEEN x AND y)
```

### Ranking Functions Cheatsheet
| Function | Ties | Gaps | Use Case |
|---|---|---|---|
| ROW_NUMBER() | No ties (unique) | N/A | Deduplication, Nth row |
| RANK() | Same rank | Yes (1,1,3) | Competition rankings |
| DENSE_RANK() | Same rank | No (1,1,2) | Top-N per group |
| NTILE(n) | N/A | N/A | Quartiles, percentiles |

### Frame Quick Reference
```sql
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW  -- running total
ROWS BETWEEN 6 PRECEDING AND CURRENT ROW          -- 7-day trailing window
ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING          -- 3-row centered window
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
