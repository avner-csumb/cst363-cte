---
# try also 'default' to start simple
theme: seriph
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
background: https://cover.sli.dev
# some information about your slides (markdown enabled)
title: CST 363
# apply UnoCSS classes to the current slide
class: text-center
# https://sli.dev/features/drawing
drawings:
  persist: false
# slide transition: https://sli.dev/guide/animations.html#slide-transitions
# transition: slide-left
# enable MDC Syntax: https://sli.dev/features/mdc
mdc: true
# duration of the presentation
duration: 35min
---

# Subqueries, CTEs, Views

CST 363


<!--
The last comment block of each slide will be treated as slide notes. It will be visible and editable in Presenter Mode along with the slide. [Read more in the docs](https://sli.dev/guide/syntax.html#notes)
-->

---
layout: section
---


## Subquery Review


---


## Definition

<div class="p-5">

- A query inside another SQL statement; enclosed in parentheses.
- Acts like a temporary table with statement scope.

<br>


```sql
SELECT city_id, city_name
FROM city
WHERE county_id IN (
    SELECT county_id
    FROM county
    WHERE county_name IN ('Monterey', 'Santa Clara')
);
```


</div>


---

## Scalar Subqueries 

<div class="p-5">


- Single row, single column (i.e., “one cell”)
- Usable with $=, <>, <, >, <=, >=$ in `WHERE` / `HAVING`

<br>


```sql
SELECT emp_id, emp_name
FROM employees
WHERE dept_id <> (
    SELECT dept_id
    FROM departments
    WHERE dept_name = 'Finance'
);
```


**Note:** $<>$ is the SQL standard for $!=$


</div>

---

## Correlated Subqueries

<div class="p-5">


- A subquery dependent on its containment statement
  - references one or more columns
- Unlike non-correlated subequery, executed for each candidate row
  - **candidate row**: row that might be included in the final results
  - non-correlated subquery is **executed once** prior to execution of containing statement

  <br>

```sql
SELECT c.first_name, c.last_name
FROM customer c
WHERE (
   SELECT COUNT(*)
   FROM orders o
   WHERE o.customer_id = c.customer_id
) > 5;
```



</div>


---


## Data Manipulation Using Correlated Subqueries

<div class="p-5">


- Subqueries are used heavily in `UPDATE`, `DELETE`, and `INSERT` statements
- Example of correlated subquery used to modify `last_update` column in `customer` table:

<br>


```sql
UPDATE customer c
SET c.last_update = (
   SELECT MAX(o.order_date)
   FROM orders o
   WHERE o.customer_id = c.customer_id
);
```

</div>

---

## Safer Update 

<div class="p-5">


- Example: Should skip customers with no orders

<br>

```sql
UPDATE customer c
SET last_update = (
   SELECT MAX(o.order_date)
   FROM orders o
   WHERE o.customer_id = c.customer_id
)
WHERE EXISTS (
   SELECT 1
   FROM orders o
   WHERE o.customer_id = c.customer_id
);
```

</div>

---

## Correlated Subquery with DELETE

<div class="p-5">


```sql
DELETE FROM customer c
WHERE (
   SELECT COALESCE(SUM(o.total_amount), 0)
   FROM orders o
   WHERE o.customer_id = c.customer_id
) < 5;
```

<br>

- If a customer has no orders, the subquery returns `NULL`
- If we just compare that to $= 0$, the condition will never match, because `NULL = 0` is unknown in SQL’s three-valued logic
- `COALESCE(expr, fallback)` replaces NULL with a fallback value.


</div>


---
layout: section
---

## Common Table Expressions (CTEs)



---

## What is a CTE (Common Table Expression)?

<div class="p-5">


- A Common Table Expression (CTE) is a temporary result set that is defined within a SQL statement using the WITH keyword.
- It makes complex queries easier to read, write, and debug.
- CTEs act like temporary named subqueries.
- They only exist for the duration of a single query (unlike views, which are permanent).
- A CTE can be referenced multiple times in the same query, reducing redundancy.


</div>

---

## CTE Keywords & Structure

<div class="p-5">

<v-clicks>

1. `WITH` $→$ Defines the CTE.
2. `AS` $→$ Assigns a name to the CTE and contains the subquery inside parentheses.
3. main query $→$ Uses the CTE in a `SELECT`, `INSERT`, `UPDATE`, or `DELETE` statement.

</v-clicks>

</div>


---

## Basic CTE Syntax

<div class="p-5">


```sql
WITH cte_name AS (
   SELECT column1, column2
   FROM some_table
   WHERE some_condition
)
SELECT * FROM cte_name;
```

<br>

- The `WITH` keyword defines a Common Table Expression (CTE).
- `cte_name` is the name of the CTE, which you can reference in the main query.

</div>


---

## CTEs vs. Subqueries

<div class="p-5">



- CTEs (Common Table Expressions) provide an alternative to subqueries.
- Advantages of CTEs over subqueries:
  - Improves readability by separating logic.
  - Can be reused multiple times within the same query.
  - Reduces deep nesting, making complex queries easier to maintain.


</div>


---

## Problem: Find all customers who have spent > $500

<div class="p-5">


Using subquery:


```sql
SELECT first_name, last_name
FROM customers
WHERE customer_id IN (
   SELECT o.customer_id
   FROM orders o
   GROUP BY o.customer_id
   HAVING SUM(o.total_amount) > 500
);
```

</div>


---

## Problem: Find all customers who have spent > $500

<div class="p-5">



With a CTE (Improved Readability)

```sql
WITH qualifying_customers AS (
   SELECT o.customer_id
   FROM orders o
   GROUP BY o.customer_id
   HAVING SUM(o.total_amount) > 500
)

SELECT c.first_name, c.last_name
FROM customers c
JOIN qualifying_customers qc
   ON c.customer_id = qc.customer_id;
```

</div>


---

## Using Multiple CTEs


<div class="p-5">

<v-clicks>

- SQL standards (and most databases) require that all CTEs for a query are declared together in one `WITH` clause, separated by commas.
- A later CTE can reference an earlier CTE 

</v-clicks>

<br>

<v-click>

```sql
WITH cte1 AS (...),
   cte2 AS (...)
SELECT ...
FROM cte1
JOIN cte2 ON ...
```

</v-click>

</div>


---
layout: section
---

## Introducing Views


---

## What is a View?

<div class="p-5">


- A **view** is a virtual table defined by a SQL query.
  - It does not store data physically; it computes results dynamically.
  - Once created, a view can be referenced by any user with appropriate permissions.
- **Benefits:**
  - Abstraction: Hide underlying table complexity.
  - Security: Limit data exposure.
  - Maintainability: Centralize logic for common queries.

</div>



---

## Creating a View in PostgreSQL

<div class="p-5">



Example: Create a view to display active employees

```sql
CREATE VIEW employee_view AS
SELECT employee_id, first_name, last_name, department
FROM employees
WHERE active = TRUE;
```

**Explanation:** This view shows only active employees, abstracting away the filtering logic.


</div>


---

## Querying a View

<div class="p-5">

Query the view just like a regular table:

```sql
SELECT * FROM employee_view;
```

- Views can simplify reporting
- Performance considerations
  - views are not indexed unless materialized

</div>


---

## Materialized Views

<div class="p-5">


- **Definition:** Similar to views but store the query’s result set physically (in a separate table-like structure) at creation or upon refresh.
- **Refresh Mechanism:** You must explicitly call `REFRESH MATERIALIZED VIEW` to update the data.

</div>
