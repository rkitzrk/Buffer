# SQL Interview Questions — Ranked by Importance for SDE Interviews

> **Priority Logic:** Coding questions first (80% weightage), then theory. Questions are ranked by frequency in real SDE interviews at Google, Amazon, Microsoft, Oracle, etc.

---

# 🔴 TOP 10 — MUST KNOW (Coding Heavy)

---

**Q1: How would you calculate the running total of sales for each product?**
*(Window Functions — #1 most asked coding question)*

**Answer (from given info):**
A running total is calculated using the SUM() window function with the OVER() clause. It adds each row's value to the cumulative total while keeping individual rows.

```sql
SELECT
    product_id,
    sale_date,
    amount,
    SUM(amount) OVER (
        PARTITION BY product_id
        ORDER BY sale_date
    ) AS running_total
FROM Sales;
```

This query calculates the cumulative sales total for each product in chronological order.

**Polished Answer:**
Running totals are a classic window function problem. The `SUM() OVER()` clause creates a cumulative sum without collapsing rows (unlike GROUP BY). The `PARTITION BY` restarts the total for each product, and `ORDER BY` ensures chronological accumulation. This is the go-to pattern for cumulative metrics, YTD totals, and running balances.

**TL;DR:** `SUM(amount) OVER (PARTITION BY product_id ORDER BY sale_date)` = running total per product.

**Key Mappings:**
- Window function → `OVER()`
- Group reset → `PARTITION BY`
- Cumulative order → `ORDER BY`
- Keeps all rows → unlike GROUP BY

---

**Q2: Explain the difference between RANK(), DENSE_RANK(), and ROW_NUMBER()**
*(Top ranking coding question)*

**Answer (from given info):**

| ROW_NUMBER() | RANK() | DENSE_RANK() |
|---|---|---|
| Assigns a unique number to each row. | Assigns the same rank to duplicate values. | Assigns the same rank to duplicate values. |
| Duplicate values receive different numbers. | Duplicate values receive the same rank. | Duplicate values receive the same rank. |
| No gaps in numbering. | Skips the next rank after duplicates. | Does not skip the next rank after duplicates. |

**Polished Answer:**
These three window functions differ in how they handle ties:
- `ROW_NUMBER()` — always unique (1,2,3,4,5), even for ties.
- `RANK()` — ties get same rank, then skips (1,2,2,4,5).
- `DENSE_RANK()` — ties get same rank, no skip (1,2,2,3,4).

Use `ROW_NUMBER()` for deduplication, `RANK()` for competitions with gaps, `DENSE_RANK()` for leaderboards without gaps.

**TL;DR:** ROW_NUMBER = no ties ever; RANK = ties + gaps; DENSE_RANK = ties + no gaps.

**Key Mappings:**
- Deduplication → ROW_NUMBER
- Olympic ranking → RANK
- Top-N per group → DENSE_RANK

---

**Q3: Explain correlated subqueries and provide an example use case**
*(High-frequency coding question)*

**Answer (from given info):**
A correlated subquery is a subquery that references columns from the outer query. It is executed once for each row processed by the outer query.

Example: Find employees whose salary is greater than the average salary of their department.

```sql
SELECT e.employee_id, e.name, e.salary, e.department_id
FROM Employees e
WHERE e.salary > (
    SELECT AVG(e2.salary)
    FROM Employees e2
    WHERE e2.department_id = e.department_id
);
```

Output: Returns employees whose salary is higher than the average salary of their department.

**Polished Answer:**
A correlated subquery depends on the outer query — it references `e.department_id` and re-executes for every outer row. Unlike a non-correlated subquery (which runs once), correlated subqueries are powerful for row-by-row comparisons. Here, each employee's salary is compared against their own department's average. Trade-off: can be slow on large tables; often rewritten as a JOIN with a derived table or window function for performance.

**TL;DR:** Correlated subquery = references outer query columns + runs per row. Classic use: compare row vs. group aggregate.

**Key Mappings:**
- Outer reference → correlation
- Runs per row → performance cost
- Alternative → JOIN + GROUP BY or window function

---

**Q4: Explain EXISTS and NOT EXISTS and how they differ from IN**
*(Very common coding + theory)*

**Answer (from given info):**
EXISTS returns TRUE if the subquery returns at least one row.
NOT EXISTS returns TRUE if the subquery returns no rows.
IN checks whether a value exists in a list or the result of a subquery.

Difference:
- EXISTS and NOT EXISTS stop searching as soon as a matching row is found and work well with correlated subqueries.
- IN compares a value against all values returned by the subquery and is best suited for small result sets.

Example:
```sql
SELECT CustomerID
FROM Customers c
WHERE EXISTS (
    SELECT 1
    FROM Orders o
    WHERE o.CustomerID = c.CustomerID
);
```
Output: Returns customers who have placed at least one order.

**Polished Answer:**
`EXISTS` is a boolean check — it short-circuits on the first match, making it fast for correlated lookups. `IN` materializes the subquery result and compares each value, which is fine for small lists but can be slow or NULL-problematic for large sets. `NOT EXISTS` is generally safer than `NOT IN` because `NOT IN` returns no rows if the subquery contains any NULL.

**TL;DR:** EXISTS = short-circuit boolean, best for correlated; IN = value list comparison, best for small static sets; NOT EXISTS > NOT IN (NULL safety).

**Key Mappings:**
- EXISTS → semi-join
- NOT EXISTS → anti-join
- IN → value list / small subquery
- NOT IN → NULL trap

---

**Q5: Explain anti-joins**
*(Common coding question)*

**Answer (from given info):**
An anti-join returns rows from one table that do not have matching rows in another table. It is commonly implemented using NOT EXISTS or a LEFT JOIN with IS NULL.

Example: Find customers who have not placed any orders.
```sql
SELECT c.CustomerID, c.CustomerName
FROM Customers c
WHERE NOT EXISTS (
    SELECT 1
    FROM Orders o
    WHERE o.CustomerID = c.CustomerID
);
```
Output: Returns customers who have not placed any orders.

**Polished Answer:**
An anti-join is the complement of a semi-join — it finds "non-matches." Two implementations:
1. `NOT EXISTS` (correlated subquery) — usually cleaner and optimizer-friendly.
2. `LEFT JOIN ... WHERE right.key IS NULL` — sometimes faster on certain engines.

Classic use cases: customers without orders, products never sold, employees without managers.

**TL;DR:** Anti-join = rows with NO match. Use NOT EXISTS or LEFT JOIN + IS NULL.

**Key Mappings:**
- Anti-join → NOT EXISTS / LEFT JOIN + IS NULL
- Semi-join → EXISTS
- Use case → "find missing" queries

---

**Q6: Explain the purpose of LAG and LEAD functions**
*(Window function coding question)*

**Answer (from given info):**
LAG and LEAD are window functions used to access values from the previous or next row without using a self-join.

- LAG() returns the value from the previous row.
- LEAD() returns the value from the next row.

They are commonly used to compare consecutive rows, calculate differences, and analyze trends.

**Polished Answer:**
`LAG()` and `LEAD()` are offset window functions. They let you peek at neighboring rows in the same result set — perfect for day-over-day comparisons, month-over-month growth, and detecting changes. Example: `LAG(sales) OVER (ORDER BY month)` gives last month's sales alongside this month's.

**TL;DR:** LAG = previous row, LEAD = next row. No self-join needed.

**Key Mappings:**
- LAG → look back
- LEAD → look forward
- Use case → consecutive row comparison

---

**Q7: How do you handle duplicates in a query without using DISTINCT?**
*(Practical coding question)*

**Answer (from given info):**
**GROUP BY:** Aggregate rows to eliminate duplicates
```sql
SELECT Column1, MAX(Column2)
FROM TableName
GROUP BY Column1;
```

**ROW_NUMBER():** Assign a unique number to each row and filter by that
```sql
WITH CTE AS (
    SELECT
        Column1,
        Column2,
        ROW_NUMBER() OVER (
            PARTITION BY Column1
            ORDER BY Column2
        ) AS RowNum
    FROM TableName
)
SELECT *
FROM CTE
WHERE RowNum = 1;
```

**Polished Answer:**
Two main approaches:
1. **GROUP BY** — collapses duplicates and lets you pick aggregates. Best when you only need grouped summaries.
2. **ROW_NUMBER() + CTE** — keeps full rows but picks one per partition. Best when you need to retain all columns and pick the "latest" or "highest" per group. This is the standard deduplication pattern in production.

**TL;DR:** GROUP BY for summaries; ROW_NUMBER() + PARTITION BY for full-row dedup.

**Key Mappings:**
- Dedup + aggregate → GROUP BY
- Dedup + keep all columns → ROW_NUMBER = 1
- Latest per group → ORDER BY date DESC

---

**Q8: What is a CTE (Common Table Expression) and when would you use it?**
*(Very common theory + coding)*

**Answer (from given info):**
A CTE (Common Table Expression) is a temporary named result set created using the WITH clause that exists only during the execution of a single SQL statement. It is used to simplify complex queries, improve readability, avoid repeating subqueries, and write recursive queries.

**Polished Answer:**
A CTE is a named temporary result set defined with `WITH`. It's like a subquery but reusable within the same statement. Use it when:
- You need the same subquery multiple times.
- You want to break a complex query into readable steps.
- You need recursion (hierarchical data).

Unlike a view, a CTE is not stored — it lives only for that one query.

**TL;DR:** CTE = `WITH name AS (...)` — temporary, readable, reusable, supports recursion.

**Key Mappings:**
- Syntax → WITH ... AS (...)
- Scope → single statement
- Recursion → WITH RECURSIVE
- Alternative → subquery / temp table / view

---

**Q9: What is the difference between UNION and UNION ALL?**
*(Must-know coding + theory)*

**Answer (from given info):**

| UNION | UNION ALL |
|---|---|
| Combines results from multiple SELECT queries and removes duplicate rows. | Combines results from multiple SELECT queries and keeps all duplicate rows. |
| Performs DISTINCT operation, so it can be slower. | Does not remove duplicates, so it is faster. |
| Used when unique results are required. | Used when all results, including duplicates, are needed. |

**Polished Answer:**
`UNION` = combine + deduplicate (implies a sort/hash step → slower). `UNION ALL` = combine only (no dedup → faster). Default to `UNION ALL` unless you specifically need uniqueness. Both require matching column count, data types, and order across the SELECTs.

**TL;DR:** UNION = dedup (slower); UNION ALL = keep all (faster). Prefer UNION ALL.

**Key Mappings:**
- UNION → DISTINCT
- UNION ALL → no dedup
- Requirement → same columns, types, order

---

**Q10: How does SQL handle recursive queries?**
*(Advanced coding — hierarchical data)*

**Answer (from given info):**
SQL uses recursive CTEs to retrieve hierarchical or tree-structured data.

```sql
WITH RecursiveCTE (ID, ParentID, Depth) AS (
    SELECT ID, ParentID, 1
    FROM Categories
    WHERE ParentID IS NULL

    UNION ALL

    SELECT c.ID, c.ParentID, r.Depth + 1
    FROM Categories c
    JOIN RecursiveCTE r
    ON c.ParentID = r.ID
)
SELECT * FROM RecursiveCTE;
```

**Polished Answer:**
A recursive CTE has two parts:
1. **Anchor member** — base case (e.g., root nodes where `ParentID IS NULL`).
2. **Recursive member** — references the CTE itself, joined via `UNION ALL`.

It keeps iterating until no new rows are produced. Classic use cases: org charts, category trees, bill-of-materials, graph traversal. Always include a depth or termination condition to avoid infinite loops.

**TL;DR:** Recursive CTE = anchor + recursive member joined by UNION ALL. For trees/hierarchies.

**Key Mappings:**
- Anchor → base case
- Recursive member → self-reference
- Termination → no new rows
- Use case → org chart, category tree

---

# 🟠 TOP 25 — CONTINUATION (Q11–Q25)

---

**Q11: What is the difference between WHERE and HAVING clauses?**
*(Top theory question)*

**Answer (from given info):**
WHERE filters individual rows before grouping or aggregation, so it can't use aggregate functions like SUM or COUNT; it's best for narrowing raw data early (e.g., a date range or status).
HAVING filters the resulting groups after GROUP BY, so it's meant for conditions on aggregates (e.g., groups with totals above a threshold).

Example:
```sql
SELECT customer_id, COUNT(*) AS orders_2025
FROM orders
WHERE order_date >= '2025-01-01' AND order_date < '2026-01-01'
GROUP BY customer_id
HAVING COUNT(*) > 5;
```

**Polished Answer:**
`WHERE` runs **before** GROUP BY — it filters raw rows and cannot use aggregates. `HAVING` runs **after** GROUP BY — it filters groups and is designed for aggregate conditions. Rule of thumb: filter early with WHERE (cheaper), filter groups with HAVING.

**TL;DR:** WHERE = row filter (pre-aggregation); HAVING = group filter (post-aggregation).

**Key Mappings:**
- WHERE → before GROUP BY, no aggregates
- HAVING → after GROUP BY, aggregates allowed
- Performance → WHERE reduces rows early

---

**Q12: What are SQL joins and the differences between INNER, LEFT, RIGHT, and FULL joins?**
*(Core theory)*

**Answer (from given info):**
SQL joins combine rows from two tables based on a matching condition (typically keys) to answer questions that span both tables.

- An INNER JOIN returns only matches that exist in both tables (the intersection).
- A LEFT JOIN returns all rows from the left table and the matching rows from the right; when there's no match, right-side columns are NULL.
- A RIGHT JOIN is the mirror image: all rows from the right table plus matches from the left, NULL when absent.
- A FULL (OUTER) JOIN returns all rows from either table, filling in NULL where a counterpart is missing.

**Polished Answer:**
Joins combine tables on a key. Visualize two overlapping circles:
- INNER = intersection only.
- LEFT = all of left + matched right.
- RIGHT = all of right + matched left.
- FULL = everything, NULLs where no match.

Most production queries use INNER and LEFT. RIGHT is rare (rewrite as LEFT by swapping tables).

**TL;DR:** INNER = match only; LEFT = all left; RIGHT = all right; FULL = all rows both sides.

**Key Mappings:**
- INNER → intersection
- LEFT → preserve left
- RIGHT → preserve right
- FULL → preserve both

---

**Q13: Describe a PRIMARY KEY and how it differs from a UNIQUE key**
*(Core theory)*

**Answer (from given info):**
A PRIMARY KEY uniquely identifies each row in a table: it combines UNIQUE + NOT NULL, there can be only one per table (though it can be composite across multiple columns), and it's the default target for foreign keys.
A UNIQUE key also enforces uniqueness, but doesn't require NOT NULL, and you can have many UNIQUE constraints per table.

**Polished Answer:**
Both enforce uniqueness, but:
- PRIMARY KEY = UNIQUE + NOT NULL, one per table, identifies rows.
- UNIQUE = uniqueness only, allows NULLs (in most DBs), many per table.

Every PRIMARY KEY is a UNIQUE key, but not vice versa. Foreign keys typically reference PRIMARY KEYs.

**TL;DR:** PK = unique + not null + one per table; UNIQUE = unique only + many per table.

**Key Mappings:**
- PK → row identity, FK target
- UNIQUE → business rule
- NULLs → PK disallows, UNIQUE allows

---

**Q14: Explain normalization and briefly describe the different normal forms**
*(Core theory)*

**Answer (from given info):**
Normalization organizes relational data to minimize redundancy and prevent update/insert/delete anomalies by splitting tables based on dependencies while preserving meaning.

- **1NF:** Each column contains atomic values, and there are no repeating groups.
- **2NF:** Meets 1NF and removes partial dependencies on a composite primary key.
- **3NF:** Meets 2NF and removes transitive dependencies.
- **BCNF:** Every determinant must be a candidate key.
- **4NF:** Removes multi-valued dependencies.
- **5NF (PJNF):** Removes join dependencies to avoid data redundancy.

**Polished Answer:**
Normalization = organizing tables to reduce redundancy and anomalies. Each normal form fixes a specific problem:
- 1NF: atomic values, no repeating groups.
- 2NF: no partial dependency on composite key.
- 3NF: no transitive dependency (non-key → non-key).
- BCNF: every determinant is a candidate key.
- 4NF: no multi-valued dependencies.
- 5NF: no join dependencies.

In practice, most OLTP schemas aim for 3NF/BCNF.

**TL;DR:** 1NF atomic → 2NF no partial → 3NF no transitive → BCNF determinant is key → 4NF multi-value → 5NF join.

**Key Mappings:**
- 1NF → atomicity
- 2NF → full dependency
- 3NF → no transitive
- BCNF → stricter 3NF

---

**Q15: How do clustered and non-clustered indexes differ?**
*(Core theory)*

**Answer (from given info):**

| Clustered Index | Non-Clustered Index |
|---|---|
| Stores table rows in the physical order of the index key. | Stores index data separately from the table with pointers to rows. |
| Only one clustered index is allowed per table. | Multiple non-clustered indexes can be created on a table. |
| Best for range queries and sorting. | Best for filtering, joins, and fast lookups. |
| Commonly created on the primary key. | Commonly created on frequently searched columns. |

**Polished Answer:**
A clustered index **is** the table — rows are physically stored in key order. Only one per table. A non-clustered index is a separate structure with pointers to rows — many per table. Clustered = great for ranges; non-clustered = great for point lookups and joins.

**TL;DR:** Clustered = physical order, one per table; Non-clustered = separate structure, many per table.

**Key Mappings:**
- Clustered → range scans, sorting
- Non-clustered → point lookups, joins
- Count → 1 clustered vs. many non-clustered

---

**Q16: How do you perform pattern matching in SQL?**
*(Common coding question)*

**Answer (from given info):**
Pattern matching in SQL is performed using the LIKE operator with wildcard characters.
- `%` matches zero or more characters.
- `_` matches exactly one character.

**Polished Answer:**
`LIKE` with wildcards:
- `'K%'` — starts with K.
- `'%K%'` — contains K.
- `'__K%'` — K at 3rd position.
- `'____'` — exactly 4 characters.

Use `NOT LIKE` for negation. For case-insensitive, use `ILIKE` (Postgres) or `LOWER()`.

**TL;DR:** LIKE + `%` (any chars) + `_` (one char).

**Key Mappings:**
- % → zero or more
- _ → exactly one
- NOT LIKE → negation

---

**Q17: What are aggregate functions in SQL?**
*(Core theory)*

**Answer (from given info):**
Aggregate functions perform calculations on a set of values and return a single value. Common aggregate functions include:
- COUNT(): Returns the number of rows.
- SUM(): Returns the total sum of values.
- AVG(): Returns the average of values.
- MIN(): Returns the smallest value.
- MAX(): Returns the largest value.

**Polished Answer:**
Aggregates collapse many rows into one value. Big five: COUNT, SUM, AVG, MIN, MAX. All ignore NULLs except `COUNT(*)`. Often paired with GROUP BY to aggregate per group and HAVING to filter groups.

**TL;DR:** COUNT/SUM/AVG/MIN/MAX — collapse rows to one value.

**Key Mappings:**
- COUNT(*) → all rows
- COUNT(col) → non-NULL only
- Others → ignore NULLs

---

**Q18: What is the purpose of the GROUP BY clause?**
*(Core theory)*

**Answer (from given info):**
The GROUP BY clause is used to arrange identical data into groups. It is typically used with aggregate functions (such as COUNT, SUM, AVG) to perform calculations on each group rather than on the entire dataset.

**Polished Answer:**
`GROUP BY` buckets rows with identical values in specified columns, then aggregates each bucket. Without it, aggregates apply to the whole table. Every non-aggregated column in SELECT must appear in GROUP BY.

**TL;DR:** GROUP BY = bucket rows → aggregate per bucket.

**Key Mappings:**
- Groups → identical values
- Pairs with → aggregates
- Rule → non-aggregated SELECT cols must be in GROUP BY

---

**Q19: What is the difference between DELETE and TRUNCATE?**
*(Very common theory)*

**Answer (from given info):**

| DELETE | TRUNCATE |
|---|---|
| Removes rows one by one, logs each deletion, allows rollback, and supports WHERE clause. | Removes all rows at once, minimal logging, faster, no rollback, and no WHERE clause. |
| It is a DML command. | It is a DDL command. |

**Polished Answer:**
`DELETE` = row-by-row, logged, rollback-able, WHERE allowed (DML). `TRUNCATE` = all rows at once, minimal logging, faster, no WHERE, no rollback (DDL). Use DELETE for selective removal; TRUNCATE to empty a table fast.

**TL;DR:** DELETE = selective + logged + rollback; TRUNCATE = all rows + fast + no rollback.

**Key Mappings:**
- DELETE → DML, WHERE allowed
- TRUNCATE → DDL, no WHERE
- Speed → TRUNCATE faster

---

**Q20: What are indexes and why are they used?**
*(Core theory)*

**Answer (from given info):**
Indexes are database objects that improve query performance by allowing faster retrieval of rows. They function like a book's index, making it quicker to find specific data without scanning the entire table. However, indexes require additional storage and can slightly slow down data modification operations.

Types of Indexes:
- **Clustered Index:** Sorts and stores data rows in order of the key (only one per table).
- **Non-Clustered Index:** Separate structure with pointers to data rows (can be many per table).
- **Unique Index:** Ensures no duplicate values.
- **Composite Index:** Index on multiple columns.

**Polished Answer:**
An index is a lookup structure (usually B-tree) that speeds reads at the cost of extra storage and slower writes. Types: clustered (physical order), non-clustered (separate + pointers), unique (enforces uniqueness), composite (multi-column). Index columns used in WHERE, JOIN, ORDER BY.

**TL;DR:** Index = faster reads, slower writes, extra storage. Index WHERE/JOIN/ORDER BY columns.

**Key Mappings:**
- Clustered → physical order
- Non-clustered → separate structure
- Composite → multi-column

---

**Q21: What are the types of constraints in SQL?**
*(Core theory)*

**Answer (from given info):**
Common constraints include:
- **NOT NULL:** Ensures a column cannot have NULL values.
- **UNIQUE:** Ensures all values in a column are distinct.
- **PRIMARY KEY:** Uniquely identifies each row in a table.
- **FOREIGN KEY:** Ensures referential integrity by linking to a primary key in another table.
- **CHECK:** Ensures that all values in a column satisfy a specific condition.
- **DEFAULT:** Sets a default value for a column when no value is specified.

**Polished Answer:**
Six constraints: NOT NULL (no nulls), UNIQUE (no duplicates), PRIMARY KEY (unique + not null), FOREIGN KEY (referential integrity), CHECK (condition), DEFAULT (fallback value). They enforce data integrity at the DB level.

**TL;DR:** NOT NULL, UNIQUE, PK, FK, CHECK, DEFAULT — DB-level data integrity.

**Key Mappings:**
- PK → UNIQUE + NOT NULL
- FK → referential integrity
- CHECK → condition

---

**Q22: How would you optimize a slow query?**
*(Practical theory)*

**Answer (from given info):**
To optimize a slow query:
- Use EXPLAIN to find slow parts.
- Add proper indexes and update statistics.
- Use efficient conditions (avoid functions on columns).
- Filter data early and avoid SELECT *.
- Optimize joins and reduce extra data.
- Rewrite queries if needed (JOIN, UNION ALL).
- Use pagination, caching, or partitioning for large data.

**Polished Answer:**
Workflow: (1) EXPLAIN to find the bottleneck. (2) Add/verify indexes on WHERE/JOIN/ORDER BY columns. (3) Avoid functions on indexed columns (kills index use). (4) Filter early with WHERE. (5) Select only needed columns. (6) Rewrite subqueries as JOINs where faster. (7) For huge tables: pagination, caching, partitioning.

**TL;DR:** EXPLAIN → index → filter early → avoid SELECT * → rewrite → paginate/partition.

**Key Mappings:**
- EXPLAIN → find bottleneck
- Index → speed lookups
- Avoid functions on indexed cols → keeps index usable

---

**Q23: What strategies can protect a web application from SQL injection?**
*(Security — frequently asked)*

**Answer (from given info):**
SQL injection can be prevented by following these best practices:
- Use parameterized queries (prepared statements).
- Validate and sanitize user input.
- Avoid dynamic SQL created through string concatenation.
- Use least-privilege database accounts.
- Use stored procedures securely.

**Polished Answer:**
#1 defense: parameterized queries / prepared statements — they separate SQL code from data. Also: validate input, never concatenate user input into SQL, use least-privilege DB accounts, and use stored procedures carefully. Defense in depth.

**TL;DR:** Parameterized queries = #1 defense. Never concatenate user input.

**Key Mappings:**
- Prepared statements → code/data separation
- Least privilege → limit damage
- Input validation → secondary defense

---

**Q24: What is a SELF JOIN and when is it used?**
*(Common coding question)*

**Answer (from given info):**
A SELF JOIN is a join in which a table is joined with itself. It is useful when rows within the same table have a relationship, such as employees and their managers or products with parent products. Separate table aliases are used to distinguish between the two instances of the same table.

```sql
SELECT e.EmployeeName,
       m.EmployeeName AS Manager
FROM Employees e
LEFT JOIN Employees m
ON e.ManagerID = m.EmployeeID;
```
Output: Returns each employee along with their manager's name.

**Polished Answer:**
A self-join joins a table to itself using two aliases. Use it for hierarchical relationships within one table: employee→manager, category→parent category, or finding pairs of rows. Always alias the two instances.

**TL;DR:** Self-join = same table, two aliases. For hierarchies and row comparisons.

**Key Mappings:**
- Aliases → distinguish instances
- Use case → employee/manager, parent/child
- Join type → usually LEFT or INNER

---

**Q25: What is the difference between CROSS JOIN and INNER JOIN?**
*(Common theory)*

**Answer (from given info):**

| CROSS JOIN | INNER JOIN |
|---|---|
| Returns the Cartesian product of both tables. | Returns only the matching rows based on a join condition. |
| Does not require a join condition. | Requires a join condition using the ON clause. |
| Every row from the first table is combined with every row from the second table. | Only rows that satisfy the join condition are returned. |
| If Table A has 3 rows and Table B has 4 rows, the result contains 12 rows (3 × 4). | The number of rows depends on the matching records between the tables. |

**Polished Answer:**
CROSS JOIN = Cartesian product (every row × every row), no ON clause. INNER JOIN = only matching rows, requires ON. CROSS JOIN is rarely used intentionally (sometimes for generating combinations); accidental CROSS JOINs are a common performance bug.

**TL;DR:** CROSS JOIN = all combinations, no condition; INNER JOIN = matches only, needs ON.

**Key Mappings:**
- CROSS → Cartesian product
- INNER → matched rows
- Accidental CROSS → performance bug

---

# 🟡 TOP 50 — CONTINUATION (Q26–Q50)

---

**Q26: What is a view in SQL?**
*(Core theory)*

**Answer (from given info):**
A view is a virtual table created from a SELECT query that displays data from one or more tables without storing it, helping simplify queries and improve security.

**Polished Answer:**
A view is a stored SELECT query that behaves like a table. It doesn't store data (unless materialized) — it runs the underlying query each time. Benefits: simplifies complex queries, hides sensitive columns, provides a stable interface over changing schemas.

**TL;DR:** View = virtual table from a SELECT. No stored data. Simplifies + secures.

**Key Mappings:**
- Virtual table → no storage
- Security → hide columns
- Stability → interface over schema

---

**Q27: What is the purpose of the UNIQUE constraint?**
*(Core theory)*

**Answer (from given info):**
The UNIQUE constraint ensures that all values in a column (or combination of columns) are distinct. This prevents duplicate values and helps maintain data integrity.

**Polished Answer:**
UNIQUE prevents duplicate values in a column or column combo. Unlike PRIMARY KEY, it allows NULLs (in most DBs) and you can have many per table. Often used for email, username, etc.

**TL;DR:** UNIQUE = no duplicates. Allows NULLs. Many per table.

**Key Mappings:**
- UNIQUE → no duplicates
- vs PK → allows NULL, many per table
- Composite → multi-column uniqueness

---

**Q28: What is a composite primary key?**
*(Core theory)*

**Answer (from given info):**
A composite primary key uses two or more columns together to uniquely identify each row when one column alone isn't sufficient.

**Polished Answer:**
A composite PK combines multiple columns to uniquely identify rows — e.g., (student_id, course_id) in an enrollment table. Each column alone may repeat, but the combination is unique.

**TL;DR:** Composite PK = multi-column uniqueness. For junction/association tables.

**Key Mappings:**
- Composite → 2+ columns
- Use case → junction tables
- Each col alone → may repeat

---

**Q29: What is a subquery?**
*(Core theory)*

**Answer (from given info):**
A subquery is a query nested within another query. It is often used in the WHERE clause to filter data based on the results of another query, making it easier to handle complex conditions.

**Polished Answer:**
A subquery is a SELECT inside another SQL statement. Can appear in WHERE, FROM, SELECT, or HAVING. Two types: non-correlated (runs once) and correlated (runs per outer row). Used to break complex logic into steps.

**TL;DR:** Subquery = query inside a query. Non-correlated (once) vs correlated (per row).

**Key Mappings:**
- Non-correlated → runs once
- Correlated → runs per row
- Positions → WHERE, FROM, SELECT, HAVING

---

**Q30: What is a query in SQL?**
*(Basic theory)*

**Answer (from given info):**
A query is a SQL statement used to retrieve, update, or manipulate data in a database. The most common type of query is a SELECT statement, which fetches data from one or more tables based on specified conditions.

**Polished Answer:**
A query is any SQL statement that interacts with data. SELECT = retrieval (DQL); INSERT/UPDATE/DELETE = modification (DML). "Query" usually implies SELECT.

**TL;DR:** Query = SQL statement. SELECT = most common.

**Key Mappings:**
- SELECT → DQL
- INSERT/UPDATE/DELETE → DML
- Query → usually means SELECT

---

**Q31: What is the difference between CHAR and VARCHAR2?**
*(Common theory)*

**Answer (from given info):**

| CHAR | VARCHAR2 |
|---|---|
| CHAR stores fixed-length character data. | VARCHAR2 stores variable-length character data. |
| It pads unused space with trailing spaces. | It does not pad unused space, saving storage. |

**Polished Answer:**
CHAR = fixed length (pads with spaces); VARCHAR2 = variable length (no padding). Use CHAR for truly fixed values (e.g., country codes); VARCHAR2 for everything else to save space.

**TL;DR:** CHAR = fixed + padded; VARCHAR2 = variable + no padding.

**Key Mappings:**
- CHAR → fixed length
- VARCHAR2 → variable length
- Storage → VARCHAR2 saves space

---

**Q32: What is the purpose of the DEFAULT constraint?**
*(Core theory)*

**Answer (from given info):**
The DEFAULT constraint assigns a default value to a column when no value is provided during an INSERT operation. This helps maintain consistent data and simplifies data entry.

**Polished Answer:**
DEFAULT provides a fallback value when INSERT omits the column. Ensures consistency and avoids NULLs where a sensible default exists (e.g., status = 'active', created_at = NOW()).

**TL;DR:** DEFAULT = fallback value on INSERT.

**Key Mappings:**
- DEFAULT → applies when omitted
- Use case → status, timestamps, flags

---

**Q33: What is denormalization and when is it used?**
*(Core theory)*

**Answer (from given info):**
Denormalization is the process of combining normalized tables into larger tables for performance reasons. It is used when complex queries and joins slow down data retrieval and the performance benefits outweigh the drawbacks of redundancy.

**Polished Answer:**
Denormalization intentionally adds redundancy to speed reads (fewer joins). Used in OLAP/data warehouses, reporting, and read-heavy systems. Trade-off: faster reads, but more complex writes and potential inconsistency.

**TL;DR:** Denormalization = add redundancy for read speed. Trade-off: complex writes.

**Key Mappings:**
- Purpose → faster reads
- Cost → redundancy, write complexity
- Use case → OLAP, reporting

---

**Q34: What is the difference between DDL and DML commands?**
*(Core theory)*

**Answer (from given info):**

| DDL (Data Definition Language) | DML (Data Manipulation Language) |
|---|---|
| Used to define and modify the structure of the database. | Used to manage and manipulate the data inside the database. |
| Works on tables, schemas, and database objects. | Works on the rows stored in tables. |
| Includes CREATE, ALTER, DROP. | Includes INSERT, UPDATE, DELETE. |
| Changes the overall structure of the database. | Changes the actual data present in the database. |

**Polished Answer:**
DDL = structure (CREATE, ALTER, DROP, TRUNCATE). DML = data (INSERT, UPDATE, DELETE, SELECT). DDL changes schema; DML changes rows. DDL is usually auto-committed; DML is transactional.

**TL;DR:** DDL = structure; DML = data.

**Key Mappings:**
- DDL → CREATE/ALTER/DROP/TRUNCATE
- DML → INSERT/UPDATE/DELETE/SELECT
- Transaction → DML rollback-able

---

**Q35: What is the purpose of the ALTER command in SQL?**
*(Core theory)*

**Answer (from given info):**
The ALTER command is used to modify the structure of an existing database object. This command is essential for adapting our database schema as requirements evolve.
- Add or drop a column in a table.
- Change a column's data type.
- Add or remove constraints.
- Rename columns or tables.
- Adjust indexing or storage settings.

**Polished Answer:**
ALTER modifies existing schema objects: add/drop columns, change types, add/remove constraints, rename. It's DDL and usually auto-commits. Use with care in production — some alters lock tables.

**TL;DR:** ALTER = change table structure (columns, types, constraints).

**Key Mappings:**
- ALTER TABLE → add/drop/modify columns
- ALTER → DDL
- Production → may lock table

---

**Q36: How is data integrity maintained in SQL databases?**
*(Core theory)*

**Answer (from given info):**
Data integrity refers to the accuracy, consistency, and reliability of data stored in the database. SQL databases maintain data integrity through:
- **Constraints:** NOT NULL, FOREIGN KEY, UNIQUE, etc.
- **Transactions:** all-or-nothing operations.
- **Triggers:** automatic rule enforcement.
- **Normalization:** minimize redundancy and anomalies.
- **Cascading actions:** ON DELETE CASCADE, ON UPDATE CASCADE.

**Polished Answer:**
Five mechanisms: constraints (NOT NULL, FK, UNIQUE, CHECK), transactions (ACID), triggers (automated rules), normalization (reduce redundancy), and cascading actions (auto propagate changes). Together they keep data accurate and consistent.

**TL;DR:** Constraints + transactions + triggers + normalization + cascades = data integrity.

**Key Mappings:**
- Constraints → rules
- Transactions → atomicity
- Triggers → automation
- Cascades → referential propagation

---

**Q37: How does the CASE statement work in SQL?**
*(Common coding question)*

**Answer (from given info):**
The CASE statement is SQL's way of implementing conditional logic in queries. It evaluates conditions and returns a value based on the first condition that evaluates to true. If no condition is met, it can return a default value using the ELSE clause.

Example:
```sql
SELECT ID,
       CASE
           WHEN Salary > 100000 THEN 'High'
           WHEN Salary BETWEEN 50000 AND 100000 THEN 'Medium'
           ELSE 'Low'
       END AS SalaryLevel
FROM Employees;
```

**Polished Answer:**
CASE = SQL's if/else. Evaluates WHEN conditions in order; returns the first match; ELSE is the fallback. Can be used in SELECT, WHERE, ORDER BY, and even inside aggregates. Essential for conditional columns and pivoting.

**TL;DR:** CASE = if/else in SQL. First WHEN wins. ELSE fallback.

**Key Mappings:**
- CASE → conditional logic
- First match → wins
- ELSE → default
- Use → SELECT, WHERE, ORDER BY

---

**Q38: What is the purpose of the COALESCE function?**
*(Common coding question)*

**Answer (from given info):**
The COALESCE function returns the first non-NULL value from a list of expressions. It's commonly used to provide default values or handle missing data gracefully.

Example:
```sql
SELECT COALESCE(NULL, NULL, 'Default Value') AS Result;
```

**Polished Answer:**
COALESCE returns the first non-NULL from its arguments. Use it to handle missing data: `COALESCE(nickname, first_name, 'Guest')`. It's the standard SQL alternative to Oracle's NVL and SQL Server's ISNULL.

**TL;DR:** COALESCE = first non-NULL value. Handles missing data.

**Key Mappings:**
- COALESCE → first non-NULL
- Use → fallback chains
- vs NVL → NVL = 2 args, COALESCE = N args

---

**Q39: What are the differences between COUNT() and SUM()?**
*(Core theory)*

**Answer (from given info):**

| COUNT() | SUM() |
|---|---|
| Counts number of rows or non-NULL values | Adds all numeric values in a column |
| `SELECT COUNT(*) FROM Orders;` | `SELECT SUM(TotalAmount) FROM Orders;` |

**Polished Answer:**
COUNT counts rows/non-NULL values; SUM adds numeric values. `COUNT(*)` counts all rows including NULLs; `COUNT(col)` counts non-NULL. SUM ignores NULLs. Both are aggregates.

**TL;DR:** COUNT = count rows; SUM = add values. Both ignore NULLs (except COUNT(*)). 

**Key Mappings:**
- COUNT(*) → all rows
- COUNT(col) → non-NULL
- SUM → total of values

---

**Q40: What are scalar functions in SQL?**
*(Core theory)*

**Answer (from given info):**
Scalar functions operate on individual values and return a single value as a result. They are often used for formatting or converting data. Common examples include:
- LEN(): Returns the length of a string.
- ROUND(): Rounds a numeric value.
- CONVERT(): Converts a value from one data type to another.

**Polished Answer:**
Scalar functions take one value and return one value (unlike aggregates). Examples: LEN (length), ROUND (rounding), CONVERT (type conversion), UPPER/LOWER, SUBSTRING. Can be used in SELECT, WHERE, ORDER BY.

**TL;DR:** Scalar = one in, one out. LEN, ROUND, CONVERT, UPPER, etc.

**Key Mappings:**
- Scalar → per-row
- Aggregate → per-group
- Examples → LEN, ROUND, CONVERT

---

**Q41: What happens if you use COUNT() on NULLs?**
*(Common theory)*

**Answer (from given info):**
COUNT(column) ignores NULL values and only counts non-NULL entries.
COUNT(*) counts all rows, including those with NULL values in columns.

**Polished Answer:**
`COUNT(col)` skips NULLs; `COUNT(*)` counts every row. This is a common interview trick — if you need total rows, use `COUNT(*)`; if you need non-NULL values in a column, use `COUNT(col)`.

**TL;DR:** COUNT(col) ignores NULLs; COUNT(*) counts all rows.

**Key Mappings:**
- COUNT(*) → all rows
- COUNT(col) → non-NULL only
- NULL → skipped by COUNT(col)

---

**Q42: What are window functions and how are they used?**
*(Core coding theory)*

**Answer (from given info):**
Window functions perform calculations across a group of related rows while keeping each row separate. They are used for tasks like running totals, rankings, and moving averages.

Example: Calculating a running total
```sql
SELECT Name, Salary,
    SUM(Salary) OVER (ORDER BY Salary) AS RunningTotal
FROM Employees;
```

**Polished Answer:**
Window functions compute across a "window" of rows but don't collapse them (unlike GROUP BY). Key clauses: `PARTITION BY` (group), `ORDER BY` (sequence), `OVER()` (window definition). Examples: ROW_NUMBER, RANK, LAG/LEAD, SUM OVER. Used for running totals, rankings, moving averages.

**TL;DR:** Window functions = per-row calculation over related rows. OVER() + PARTITION BY + ORDER BY.

**Key Mappings:**
- OVER() → defines window
- PARTITION BY → group reset
- ORDER BY → row sequence

---

**Q43: What is the difference between an index and a key?**
*(Core theory)*

**Answer (from given info):**

| Index | Key |
|---|---|
| Used to improve data retrieval speed | Used to enforce data integrity and relationships |
| Physical database object | Logical concept |
| Does not ensure uniqueness (can be non-unique) | Ensures uniqueness (e.g., Primary Key) |
| Helps in faster searching of data | Defines relationships (e.g., Foreign Key) |

**Polished Answer:**
A key is a logical constraint (PK, FK, UNIQUE) that enforces integrity. An index is a physical structure for speed. Keys are often implemented with indexes (e.g., PK creates a clustered index), but indexes can exist without being keys (non-unique indexes).

**TL;DR:** Key = logical integrity; Index = physical speed. PK → usually clustered index.

**Key Mappings:**
- Key → integrity
- Index → performance
- PK → index-backed

---

**Q44: How does indexing improve query performance?**
*(Core theory)*

**Answer (from given info):**
Indexing helps the database quickly find data without scanning the whole table, reducing time and improving query performance.

Example:
```sql
CREATE INDEX idx_lastname ON Employees(LastName);
SELECT * FROM Employees WHERE LastName = 'Smith';
```
The index on LastName lets the database quickly find all rows matching 'Smith' without scanning every record.

**Polished Answer:**
An index is like a book's index — instead of scanning every page (full table scan), the DB looks up the value in the index (B-tree) and jumps to matching rows. Dramatically speeds WHERE, JOIN, and ORDER BY on indexed columns.

**TL;DR:** Index → avoid full table scan → faster lookups.

**Key Mappings:**
- Index → B-tree lookup
- Without index → full scan
- Speeds → WHERE, JOIN, ORDER BY

---

**Q45: What are the trade-offs of using indexes?**
*(Core theory)*

**Answer (from given info):**
**Advantages:**
- Faster query performance, especially for SELECT with WHERE, JOIN, ORDER BY.
- Improved sorting and filtering efficiency.

**Disadvantages:**
- Increased storage space for index structures.
- Additional overhead for write operations (INSERT, UPDATE, DELETE), as indexes must be updated.

**Polished Answer:**
Indexes speed reads but cost storage and slow writes. Every INSERT/UPDATE/DELETE must also update the index. Too many indexes = write bottleneck. Balance: index what you query, not everything.

**TL;DR:** Indexes: faster reads, slower writes, more storage.

**Key Mappings:**
- Read → faster
- Write → slower
- Storage → extra

---

**Q46: What is a materialized view and how does it differ from a standard view?**
*(Advanced theory)*

**Answer (from given info):**
**Standard View:**
- A virtual table defined by a query.
- Does not store data; the underlying query is executed each time the view is referenced.
- Shows real-time data.

**Materialized View:**
- A physical table that stores the result of the query.
- Data is precomputed and stored, making reads faster.
- Requires periodic refreshes to keep data up to date.
- Used to store aggregated sales data, updated nightly, for fast reporting.

Refresh:
```sql
REFRESH MATERIALIZED VIEW my_view;
```

**Polished Answer:**
A standard view runs its query every time (always fresh, slower). A materialized view stores the result physically (fast reads, but stale until refreshed). Use materialized views for expensive aggregations in reporting/OLAP; standard views for simplicity and security.

**TL;DR:** View = virtual, always fresh; Materialized view = stored, fast but stale until refresh.

**Key Mappings:**
- View → virtual, live
- Materialized → stored, needs refresh
- Use case → reporting aggregations

---

**Q47: What are the ACID properties of a transaction?**
*(Core theory — very important)*

**Answer (from given info):**
ACID ensures database transactions are reliable and maintain data integrity.
- **Atomicity:** A transaction is completed entirely or not at all. If one operation fails, all changes are rolled back.
- **Consistency:** Ensures the database remains valid by following all rules and constraints.
- **Isolation:** Multiple transactions execute independently without affecting each other.
- **Durability:** Once a transaction is committed, its changes are permanently saved, even after a system failure.

**Polished Answer:**
ACID = Atomicity (all or nothing), Consistency (valid state transitions), Isolation (concurrent transactions don't interfere), Durability (committed = permanent). These guarantee reliable transactions in relational DBs. Example: bank transfer must debit and credit atomically.

**TL;DR:** ACID = Atomicity + Consistency + Isolation + Durability.

**Key Mappings:**
- Atomicity → all or nothing
- Consistency → rules preserved
- Isolation → concurrent safety
- Durability → committed = permanent

---

**Q48: What are the differences between isolation levels in SQL?**
*(Core theory)*

**Answer (from given info):**
Isolation levels control how transactions interact to maintain data consistency.
- **Read Uncommitted:** Allows reading uncommitted data → dirty reads.
- **Read Committed:** Reads only committed data → prevents dirty reads.
- **Repeatable Read:** Same data remains unchanged during a transaction → prevents dirty and non-repeatable reads.
- **Serializable:** Highest isolation → prevents dirty, non-repeatable, and phantom reads.

**Polished Answer:**
Four levels, increasing strictness:
1. Read Uncommitted — dirty reads possible.
2. Read Committed — no dirty reads (default in many DBs).
3. Repeatable Read — no non-repeatable reads.
4. Serializable — no phantom reads (fully serial).

Higher isolation = more consistency, less concurrency.

**TL;DR:** Read Uncommitted → Read Committed → Repeatable Read → Serializable. More isolation = less concurrency.

**Key Mappings:**
- Dirty read → prevented by Read Committed
- Non-repeatable → prevented by Repeatable Read
- Phantom → prevented by Serializable

---

**Q49: What are the differences between OLTP and OLAP systems?**
*(Core theory)*

**Answer (from given info):**

| OLTP | OLAP |
|---|---|
| Handles simple, frequent transactions | Handles complex queries and data analysis |
| Optimized for fast read/write | Optimized for read-heavy workloads and aggregation |
| Normalized schema | Denormalized schema (star/snowflake) |
| Example: E-commerce, banking | Example: Data warehousing, BI |

**Polished Answer:**
OLTP = day-to-day transactions (INSERT/UPDATE/DELETE), normalized, high concurrency, low latency. OLAP = analytics (complex SELECTs, aggregations), denormalized (star/snowflake), read-heavy, large scans. Different workloads → different designs.

**TL;DR:** OLTP = transactions, normalized, fast writes; OLAP = analytics, denormalized, fast reads.

**Key Mappings:**
- OLTP → normalized, concurrent writes
- OLAP → star schema, aggregations
- Use case → apps vs. BI

---

**Q50: What is database sharding and how does it differ from partitioning?**
*(Advanced theory)*

**Answer (from given info):**

| Sharding | Partitioning |
|---|---|
| Splits database into multiple independent databases | Splits a table into parts within the same database |
| Used for horizontal scaling across servers | Used for better performance and data management |
| Data is stored on different servers | Data stays in one database |
| Example: Database divided by region | Example: Table divided by year |

**Polished Answer:**
Partitioning splits a table within one database (e.g., by year) — improves manageability and query pruning. Sharding splits data across multiple independent databases/servers (e.g., by region) — enables horizontal scale. Partitioning is a single-DB optimization; sharding is a distributed-system architecture.

**TL;DR:** Partitioning = split table in one DB; Sharding = split across multiple DBs/servers.

**Key Mappings:**
- Partitioning → single DB, performance
- Sharding → multiple DBs, horizontal scale
- Use case → large table vs. global app

---

# 📊 SUMMARY TABLE

| Rank | Question | Type | Frequency |
|------|----------|------|-----------|
| 1 | Running Total (Window Function) | Coding | ⭐⭐⭐⭐⭐ |
| 2 | RANK vs DENSE_RANK vs ROW_NUMBER | Coding | ⭐⭐⭐⭐⭐ |
| 3 | Correlated Subqueries | Coding | ⭐⭐⭐⭐⭐ |
| 4 | EXISTS vs IN | Coding | ⭐⭐⭐⭐⭐ |
| 5 | Anti-Joins | Coding | ⭐⭐⭐⭐ |
| 6 | LAG and LEAD | Coding | ⭐⭐⭐⭐ |
| 7 | Deduplication without DISTINCT | Coding | ⭐⭐⭐⭐ |
| 8 | CTE | Coding + Theory | ⭐⭐⭐⭐⭐ |
| 9 | UNION vs UNION ALL | Coding + Theory | ⭐⭐⭐⭐⭐ |
| 10 | Recursive Queries | Coding | ⭐⭐⭐⭐ |
| 11 | WHERE vs HAVING | Theory | ⭐⭐⭐⭐⭐ |
| 12 | SQL Joins (INNER/LEFT/RIGHT/FULL) | Theory | ⭐⭐⭐⭐⭐ |
| 13 | PRIMARY KEY vs UNIQUE | Theory | ⭐⭐⭐⭐⭐ |
| 14 | Normalization (1NF–5NF) | Theory | ⭐⭐⭐⭐ |
| 15 | Clustered vs Non-Clustered Index | Theory | ⭐⭐⭐⭐⭐ |
| 16 | Pattern Matching (LIKE) | Coding | ⭐⭐⭐⭐ |
| 17 | Aggregate Functions | Theory | ⭐⭐⭐⭐ |
| 18 | GROUP BY | Theory | ⭐⭐⭐⭐ |
| 19 | DELETE vs TRUNCATE | Theory | ⭐⭐⭐⭐ |
| 20 | Indexes (Types & Purpose) | Theory | ⭐⭐⭐⭐ |
| 21 | Constraints | Theory | ⭐⭐⭐ |
| 22 | Query Optimization | Theory | ⭐⭐⭐⭐ |
| 23 | SQL Injection Prevention | Security | ⭐⭐⭐⭐ |
| 24 | SELF JOIN | Coding | ⭐⭐⭐ |
| 25 | CROSS JOIN vs INNER JOIN | Theory | ⭐⭐⭐ |
| 26–50 | View, CHAR/VARCHAR2, DDL/DML, ACID, OLTP/OLAP, Sharding, etc. | Theory | ⭐⭐⭐ |

---

**Final Note:** For SDE interviews, **coding questions (Q1–Q10)** are the highest priority — practice writing them by hand or in a SQL editor until fluent. **Q11–Q25** are core theory that appears in almost every interview. **Q26–Q50** are important but less frequently asked; review them to round out your knowledge.
