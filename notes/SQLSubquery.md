# Question

What exactly will this return?
A) Highest paid employee only
B) Employees earning above average salary C) Employees with duplicate salaries
D) All employees

What's the correct answer?
SELECT * FROM employees
WHERE salary > (
SELECT AVG(salary)
FROM employees
);

# Answer

# The Big Picture & Analogy

Imagine a teacher gives students a test, then says: **"Anyone who scored above the class average, raise your hand."**

The teacher didn't say "whoever got the single highest score" (that would only be one person), and didn't say "everyone raise your hand" (that has no condition at all) — the teacher set a middle benchmark, **"the average,"** and compared everyone against it. This might result in several people raising their hands (everyone above average), or possibly no one at all (if everyone scored exactly the same, e.g. everyone got 50, the average is exactly 50, and nobody is "above" it).

# Why Do We Need It? (Understanding This Subquery Pattern Correctly)

This query has **two parts, each doing a different job**:

```sql
SELECT * FROM employees
WHERE salary > (
    SELECT AVG(salary) 
    FROM employees
);
```

**Part 1 (Inner Query / Subquery):**
```sql
SELECT AVG(salary) FROM employees
```
Computes a **single scalar value** — "the average salary of all employees." Say there are 5 employees earning 30k, 40k, 50k, 60k, 70k → the average is **50k**.

**Part 2 (Outer Query):**
```sql
SELECT * FROM employees WHERE salary > 50000
```
Takes the 50k value from the subquery as the threshold, then filters for **anyone earning more than 50k** — in this example, that returns the employees earning 60k and 70k (2 people).

Note that this is **not a correlated subquery** (one that references the outer query's current row). This is a **non-correlated subquery** — it computes a single value once, then applies that same condition to every row in the outer query.

# Core Logic & How It Works (Analyzing Each Option)

**❌ A) Highest paid employee only**
Wrong — to get "just the single highest-paid employee," you'd write:
```sql
SELECT * FROM employees ORDER BY salary DESC LIMIT 1;
```
The query in the question has no `ORDER BY` or `LIMIT` at all — it filters using "greater than average," which could match multiple rows, not just one.

**✅ B) Employees earning above average salary**
Correct — this query computes the average first, then filters for **everyone whose salary exceeds it**, which could be 0, 1, or many people depending on the actual data distribution.

**❌ C) Employees with duplicate salaries**
Wrong — finding "duplicate salaries" requires `GROUP BY` with `HAVING COUNT(*) > 1`:
```sql
SELECT salary FROM employees GROUP BY salary HAVING COUNT(*) > 1;
```
The query in question has no concept of "counting duplicates" at all.

**❌ D) All employees**
Wrong — there's a `WHERE` clause, which filters data by definition; it's not a query that returns every row unconditionally (to get everyone, you'd need no `WHERE` at all, or `WHERE 1=1`).

# Trade-offs & When to Use (Performance Perspective for a Lead)

**Key trade-off with this pattern:** this subquery gets **re-evaluated on each execution** (results aren't cached across runs). But since it's a **non-correlated subquery** (it doesn't reference the outer query's rows), most database optimizers (PostgreSQL, MySQL) are smart enough to **compute the average just once**, then compare it against every row — unlike a **correlated subquery**, which genuinely re-executes per row and is much slower.

**When to watch out:** if the table has a huge number of rows (millions), computing `AVG()` on every query execution can carry real cost — if this query runs frequently, consider caching the average value (e.g. in Redis) and invalidating it whenever salaries change, rather than computing it fresh every time.

**Alternative achieving the same result with different syntax (Window Function):**
```sql
SELECT * FROM (
    SELECT *, AVG(salary) OVER () as avg_salary
    FROM employees
) t
WHERE salary > avg_salary;
```
Some database engines optimize this better, since the average is computed once via a window function and attached to every row directly.

# Real-World Scenario (Ecommerce Domain)

This pattern is extremely common for finding **"customers spending above average"** (VIP customer detection):

```sql
SELECT customer_id, SUM(total_amount) as total_spent
FROM orders
GROUP BY customer_id
HAVING SUM(total_amount) > (
    SELECT AVG(customer_total) 
    FROM (
        SELECT customer_id, SUM(total_amount) as customer_total
        FROM orders
        GROUP BY customer_id
    ) t
);
```
Exact same logic — compute "average spend per customer" first, then filter for anyone spending more than that, to target marketing campaigns or give perks to a VIP segment.

# Lead's Key Takeaway

> **"When reading an SQL subquery, always parse it layer by layer: what does the inner layer compute (a single value or many values), and what does the outer layer compare it against — don't try to read the whole query in one pass and get overwhelmed."**
>
> A good Lead should be able to read SQL well enough to instantly distinguish correlated from non-correlated subqueries, because it has a **direct performance impact** — a correlated subquery that looks similar on the surface but re-executes per row can turn an innocent-looking query into a guaranteed slow-query incident in production.
