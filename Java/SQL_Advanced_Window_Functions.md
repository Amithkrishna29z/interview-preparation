# SQL Advanced Window Functions & Query Optimization — Awareness Notes

> **Scope note (junior job prep):** Advanced SQL (window functions, frame specs, recursive CTEs, LATERAL joins, advanced aggregation) is **deferred for later study**. This file is trimmed to a short awareness section. Your junior-level SQL essentials — CRUD, joins, aggregates/GROUP BY, subqueries, `EXPLAIN`, indexing, transactions, JSON — are covered in **`Database_Concepts_Interview_Questions.md`**, **`PostgreSQL_Interview_Questions.md`**, and **`MySQL_Interview_Questions.md`**, which are kept. The full advanced-SQL deep-dive remains in git history.

---

## The One Concept Worth Recognizing

**Window functions** compute a value across a set of rows *related to the current row* — without collapsing them like `GROUP BY` does. Think "an aggregate that keeps every row."

```sql
-- GROUP BY: 10 rows → 3 rows (one per department)
SELECT department_id, AVG(salary) FROM employees GROUP BY department_id;

-- Window function: keeps all 10 rows, adds the dept average as a column
SELECT name, department_id, salary,
       AVG(salary) OVER (PARTITION BY department_id) AS dept_avg
FROM employees;
```

- Common ones to recognize by name: `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()` (ranking), `LAG()`/`LEAD()` (previous/next row), and `SUM() OVER (...)` (running totals).
- `PARTITION BY` restarts the calculation per group; `ORDER BY` inside `OVER(...)` orders rows within the window.
- **Basic CTEs** (`WITH name AS (SELECT ...)`) are worth knowing — they just name a subquery to make complex queries readable. Recursive CTEs (hierarchies) are advanced.

> **Interview soundbite:** "I know window functions are aggregates that keep each row — `ROW_NUMBER`, `RANK`, running totals with `OVER (PARTITION BY ...)`. I'm solid on joins, GROUP BY, and subqueries; the deeper window-frame and recursive-CTE stuff I'd reach for the docs on."

---

*Trimmed to awareness level for junior job prep. Restore the full advanced-SQL deep-dive from version control when you're ready to study it.*
