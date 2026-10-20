# SQL — Coding Questions

Classic SQL coding questions asked in interviews, with working Postgres-flavored SQL and a plain-language walkthrough of the logic (not just the syntax), since that's what survives a "now modify it to find the 3rd highest instead" follow-up.

Sample table used throughout:
```sql
CREATE TABLE employee (
    id INT PRIMARY KEY,
    name TEXT,
    department TEXT,
    salary NUMERIC,
    manager_id INT
);
```

## Q1: Find the Nth highest salary (e.g. 2nd highest)
```sql
-- Approach 1: subquery with DISTINCT + LIMIT/OFFSET
SELECT DISTINCT salary
FROM employee
ORDER BY salary DESC
LIMIT 1 OFFSET 1;   -- OFFSET 1 skips the highest, giving the 2nd highest

-- Approach 2: window function (better when you need this per-group, or want to avoid LIMIT/OFFSET)
SELECT salary FROM (
    SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
    FROM employee
) ranked
WHERE rnk = 2;
```
**Logic in plain words:** Approach 1 — sort salaries highest-first, remove duplicates (`DISTINCT`, so two people tied for #1 don't push the real 2nd place out), then skip the first row (`OFFSET 1`) and take the next one (`LIMIT 1`). Approach 2 — `DENSE_RANK()` assigns rank 1 to the highest, rank 2 to the next *distinct* value (ties share a rank, and the next distinct value still gets the very next rank number — unlike `RANK()`, which would leave a gap after a tie). Filtering `WHERE rnk = 2` is then just "give me whoever's in 2nd place" — and changing `rnk = 2` to `rnk = 3` is the entire change needed for "3rd highest," which is why interviewers like this version better than the `LIMIT`/`OFFSET` one.

---

## Q2: Highest salary in each department
```sql
SELECT department, MAX(salary) AS highest_salary
FROM employee
GROUP BY department;

-- If you also need the employee's NAME, not just the number:
SELECT department, name, salary FROM (
    SELECT department, name, salary,
           RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS rnk
    FROM employee
) ranked
WHERE rnk = 1;
```
**Logic in plain words:** the first query alone can tell you the highest *number* per department, but can't also give you *whose* salary that was in the same simple query (that's what `GROUP BY` fundamentally can't do — it collapses rows). `PARTITION BY department` in the window-function version is the key idea: it's like running `ORDER BY salary DESC` *separately within each department*, resetting the ranking back to 1 every time the department changes — so `rnk = 1` gives you the top earner's full row, per department, in one query.

---

## Q3: Employees who earn more than their manager (self-join)
```sql
SELECT e.name AS employee, e.salary, m.name AS manager, m.salary AS manager_salary
FROM employee e
JOIN employee m ON e.manager_id = m.id
WHERE e.salary > m.salary;
```
**Logic in plain words:** the same table is joined to itself twice — once playing the role of "the employee" (`e`), once playing the role of "that employee's manager" (`m`), matched via `e.manager_id = m.id`. Once joined, each row has both salaries side by side, so the filter is a plain comparison. This self-join shape (one table, two aliases, joined on a "points to another row in the same table" column) is the general pattern for any hierarchical/self-referencing relationship, not just manager/employee.

---

## Q4: Find duplicate rows (same values across multiple columns)
```sql
SELECT name, department, salary, COUNT(*) AS occurrences
FROM employee
GROUP BY name, department, salary
HAVING COUNT(*) > 1;
```
**Logic in plain words:** `GROUP BY` collapses every row with the same `(name, department, salary)` combination into one group; `COUNT(*)` counts how many original rows fell into that group. `HAVING COUNT(*) > 1` filters groups *after* grouping — this has to be `HAVING`, not `WHERE`, because `WHERE` filters individual rows *before* grouping happens, and "this row is a duplicate" is a property of the whole group, not of any single row in isolation.

---

## Q5: Delete duplicate rows, keeping exactly one copy of each
```sql
DELETE FROM employee
WHERE id NOT IN (
    SELECT MIN(id)
    FROM employee
    GROUP BY name, department, salary
);
```
**Logic in plain words:** for every group of duplicates (same `name, department, salary`), decide to keep just one — here, the row with the smallest `id` in that group (`MIN(id)` per group, via the same `GROUP BY` idea as Q4). Then delete every row whose `id` is *not* one of those "keep" ids. The subquery produces exactly one surviving `id` per duplicate-group; everything else matching that group's values gets removed.

---

## Q6: Running total (cumulative sum) of a value, ordered by date
```sql
SELECT trade_date, amount,
       SUM(amount) OVER (ORDER BY trade_date) AS running_total
FROM trades;
```
**Logic in plain words:** a plain `SUM(amount)` with `GROUP BY` would collapse all rows into one total — not what's wanted here. `SUM(amount) OVER (ORDER BY trade_date)` is a **window function**: it computes a sum for each row individually, using a "window" of rows defined by the `OVER` clause — by default, `ORDER BY` inside `OVER` (with no explicit frame) means "every row up to and including this one, in date order." So row 3's `running_total` is the sum of rows 1, 2, and 3's amounts — exactly a running total, without collapsing the individual rows the way `GROUP BY` would.

**How this maps to your project:** this is directly useful for a position/exposure report in `settlement-service` — a running balance per account as trades settle over time, ordered by `trade_date`.

---

## Q7: Rank employees within each department — RANK() vs DENSE_RANK() vs ROW_NUMBER()
```sql
SELECT name, department, salary,
       RANK()       OVER (PARTITION BY department ORDER BY salary DESC) AS rnk,
       DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dense_rnk,
       ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS row_num
FROM employee;
```
**Logic in plain words — how they differ on a tie** (say two people in Engineering are tied for highest salary):
- **`ROW_NUMBER()`** gives every row a unique number regardless of ties (1, 2, 3, 4...) — it doesn't care that two rows have the same salary; it just needs some deterministic order.
- **`RANK()`** gives tied rows the *same* rank, but then **skips** the next rank number (1, 1, 3, 4 — rank 2 is skipped because two people share rank 1).
- **`DENSE_RANK()`** also gives tied rows the same rank, but does **not** skip the next number (1, 1, 2, 3 — the next distinct salary is immediately rank 2).

**Which to use when:** `ROW_NUMBER()` when you need exactly one row per "slot" regardless of ties (e.g. "give me exactly the top 1 row per department" — see Q2's version using `RANK() = 1`, which would wrongly return *two* rows for a department with a tied top salary; `ROW_NUMBER() = 1` would correctly return exactly one). `DENSE_RANK()` when ties should share a position but you don't want gaps in the numbering (this is exactly Q1's "Nth highest" use, since a gap from `RANK()` would make "2nd highest" skip past a legitimately-tied value).

---

## Q8: Count employees per department, including departments with zero employees
```sql
SELECT d.department_name, COUNT(e.id) AS employee_count
FROM department d
LEFT JOIN employee e ON e.department = d.department_name
GROUP BY d.department_name;
```
**Logic in plain words:** an `INNER JOIN` here would silently drop any department that has no matching employee rows at all — it only keeps rows with a match on both sides. `LEFT JOIN` keeps every row from `department` (the left table) regardless of whether a match exists, filling in `NULL` for the employee side when there's no match. `COUNT(e.id)` — specifically counting the employee's `id` column, not `COUNT(*)` — is important here too: `COUNT(*)` would count the one `NULL`-filled placeholder row as "1" for an empty department, while `COUNT(e.id)` correctly counts zero, because `COUNT` of a column ignores `NULL`s.
