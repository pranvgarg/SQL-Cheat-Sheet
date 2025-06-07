# The Complete SQL Interview Preparation Guide

This comprehensive guide covers essential SQL concepts commonly tested in coding interviews. Each section includes clear explanations, practical examples, and professional tips to help you excel in technical interviews.

---

## Table of Contents
1. [JOINs - The Foundation of Relational Queries](#1-joins---the-foundation-of-relational-queries)
2. [Subqueries & CTEs - Advanced Query Composition](#2-subqueries--ctes---advanced-query-composition)
3. [Set Operators - Combining Result Sets](#3-set-operators---combining-result-sets)
4. [Advanced Filtering - WHERE vs HAVING](#4-advanced-filtering---where-vs-having)
5. [NULL Handling - Managing Missing Data](#5-null-handling---managing-missing-data)
6. [SQL Order of Operations - Understanding Query Execution](#6-sql-order-of-operations---understanding-query-execution)
7. [Window Functions - Advanced Analytics](#7-window-functions---advanced-analytics)
8. [Datetime Functions - Time Manipulation](#8-datetime-functions---time-manipulation)
9. [String Functions - Text Processing](#9-string-functions---text-processing)
10. [Aggregation & Grouping - Data Summarization](#10-aggregation--grouping---data-summarization)
11. [Advanced Concepts - Professional Techniques](#11-advanced-concepts---professional-techniques)

---

## 1. JOINs - The Foundation of Relational Queries

JOINs are the most tested SQL concept in interviews. Understanding different types and when to use them is crucial.

### 1.1 INNER JOIN
Returns only rows that have matching values in both tables.

**Syntax:**
```sql
SELECT columns
FROM table1 t1
INNER JOIN table2 t2 ON t1.column = t2.column;
```

**Example:** Find all orders with customer information
```sql
SELECT 
    o.order_id,
    o.order_date,
    c.customer_name,
    c.email
FROM orders o
INNER JOIN customers c ON o.customer_id = c.customer_id;
```

**Pro Tip:** INNER JOIN is the default - you can omit the "INNER" keyword.

### 1.2 LEFT JOIN (LEFT OUTER JOIN)
Returns all rows from the left table and matching rows from the right table. NULL values for non-matching right table columns.

**Example:** Find all customers and their orders (including customers with no orders)
```sql
SELECT 
    c.customer_name,
    o.order_id,
    o.order_date
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
ORDER BY c.customer_name;
```

**Interview Question:** "Show customers who haven't placed any orders"
```sql
SELECT c.customer_name
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE o.customer_id IS NULL;
```

### 1.3 RIGHT JOIN (RIGHT OUTER JOIN)
Returns all rows from the right table and matching rows from the left table.

**Example:** Find all products and their order details (including products never ordered)
```sql
SELECT 
    p.product_name,
    od.quantity,
    od.unit_price
FROM order_details od
RIGHT JOIN products p ON od.product_id = p.product_id;
```

### 1.4 FULL OUTER JOIN
Returns all rows when there's a match in either table.

**Example:** Complete view of customers and orders
```sql
SELECT 
    c.customer_name,
    o.order_id
FROM customers c
FULL OUTER JOIN orders o ON c.customer_id = o.customer_id;
```

**Note:** MySQL doesn't support FULL OUTER JOIN directly. Use UNION of LEFT and RIGHT JOINs.

### 1.5 SELF JOIN
Joining a table with itself, useful for hierarchical data.

**Example:** Find employees and their managers
```sql
SELECT 
    e.employee_name AS employee,
    m.employee_name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.employee_id;
```

**Interview Question:** "Find employees who earn more than their managers"
```sql
SELECT 
    e.employee_name,
    e.salary AS employee_salary,
    m.employee_name AS manager,
    m.salary AS manager_salary
FROM employees e
INNER JOIN employees m ON e.manager_id = m.employee_id
WHERE e.salary > m.salary;
```

### 1.6 CROSS JOIN
Cartesian product of two tables. Use with caution!

**Example:** Generate all possible product-category combinations
```sql
SELECT 
    p.product_name,
    c.category_name
FROM products p
CROSS JOIN categories c;
```

---

## 2. Subqueries & CTEs - Advanced Query Composition

### 2.1 Scalar Subqueries
Returns a single value, used in SELECT, WHERE, or HAVING clauses.

**Example:** Find products priced above average
```sql
SELECT 
    product_name,
    price
FROM products
WHERE price > (SELECT AVG(price) FROM products);
```

### 2.2 Row Subqueries
Returns a single row with multiple columns.

**Example:** Find the customer who placed the most recent order
```sql
SELECT customer_name
FROM customers
WHERE (customer_id, created_date) = (
    SELECT customer_id, MAX(order_date)
    FROM orders
    GROUP BY customer_id
    ORDER BY MAX(order_date) DESC
    LIMIT 1
);
```

### 2.3 Table Subqueries
Returns multiple rows and columns.

**Example:** Find customers in cities with more than 5 customers
```sql
SELECT customer_name, city
FROM customers
WHERE city IN (
    SELECT city
    FROM customers
    GROUP BY city
    HAVING COUNT(*) > 5
);
```

### 2.4 Correlated Subqueries
References columns from the outer query.

**Example:** Find customers who have placed above-average orders
```sql
SELECT 
    c.customer_name,
    c.customer_id
FROM customers c
WHERE (
    SELECT AVG(o.total_amount)
    FROM orders o
    WHERE o.customer_id = c.customer_id
) > (
    SELECT AVG(total_amount) FROM orders
);
```

### 2.5 EXISTS vs IN
**EXISTS** is often more efficient for checking existence.

**Example:** Find customers who have placed orders (using EXISTS)
```sql
SELECT customer_name
FROM customers c
WHERE EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.customer_id
);
```

**Example:** Same query using IN
```sql
SELECT customer_name
FROM customers
WHERE customer_id IN (
    SELECT DISTINCT customer_id
    FROM orders
    WHERE customer_id IS NOT NULL
);
```

### 2.6 Common Table Expressions (CTEs)
Named temporary result sets that improve readability.

**Example:** Recursive CTE for organizational hierarchy
```sql
WITH RECURSIVE employee_hierarchy AS (
    -- Base case: Top-level managers
    SELECT 
        employee_id,
        employee_name,
        manager_id,
        0 AS level,
        employee_name AS path
    FROM employees
    WHERE manager_id IS NULL
    
    UNION ALL
    
    -- Recursive case: Employees with managers
    SELECT 
        e.employee_id,
        e.employee_name,
        e.manager_id,
        eh.level + 1,
        CONCAT(eh.path, ' -> ', e.employee_name)
    FROM employees e
    INNER JOIN employee_hierarchy eh ON e.manager_id = eh.employee_id
)
SELECT * FROM employee_hierarchy
ORDER BY level, employee_name;
```

**Interview Question:** "Find the top 3 customers by total order value"
```sql
WITH customer_totals AS (
    SELECT 
        c.customer_id,
        c.customer_name,
        SUM(o.total_amount) AS total_spent
    FROM customers c
    INNER JOIN orders o ON c.customer_id = o.customer_id
    GROUP BY c.customer_id, c.customer_name
)
SELECT 
    customer_name,
    total_spent,
    RANK() OVER (ORDER BY total_spent DESC) AS spending_rank
FROM customer_totals
ORDER BY total_spent DESC
LIMIT 3;
```

---

## 3. Set Operators - Combining Result Sets

### 3.1 UNION
Combines results from two queries, removing duplicates.

**Example:** Get all product names from both active and discontinued products
```sql
SELECT product_name, 'Active' AS status
FROM active_products
UNION
SELECT product_name, 'Discontinued' AS status
FROM discontinued_products
ORDER BY product_name;
```

### 3.2 UNION ALL
Combines results keeping all duplicates (faster than UNION).

**Example:** Combine all transactions from different years
```sql
SELECT transaction_id, amount, transaction_date
FROM transactions_2023
UNION ALL
SELECT transaction_id, amount, transaction_date
FROM transactions_2024
ORDER BY transaction_date;
```

### 3.3 INTERSECT
Returns common rows between two result sets.

**Example:** Find customers who bought both electronics and books
```sql
SELECT customer_id
FROM orders o
INNER JOIN order_details od ON o.order_id = od.order_id
INNER JOIN products p ON od.product_id = p.product_id
WHERE p.category = 'Electronics'

INTERSECT

SELECT customer_id
FROM orders o
INNER JOIN order_details od ON o.order_id = od.order_id
INNER JOIN products p ON od.product_id = p.product_id
WHERE p.category = 'Books';
```

### 3.4 EXCEPT (MINUS in Oracle)
Returns rows from the first query that don't exist in the second.

**Example:** Find customers who bought electronics but not books
```sql
SELECT customer_id
FROM orders o
INNER JOIN order_details od ON o.order_id = od.order_id
INNER JOIN products p ON od.product_id = p.product_id
WHERE p.category = 'Electronics'

EXCEPT

SELECT customer_id
FROM orders o
INNER JOIN order_details od ON o.order_id = od.order_id
INNER JOIN products p ON od.product_id = p.product_id
WHERE p.category = 'Books';
```

---

## 4. Advanced Filtering - WHERE vs HAVING

Understanding when to use WHERE vs HAVING is a classic interview question.

### 4.1 WHERE Clause
Filters rows **before** grouping occurs.

**Example:** Find average salary by department, only for employees hired after 2020
```sql
SELECT 
    department,
    AVG(salary) AS avg_salary
FROM employees
WHERE hire_date > '2020-01-01'  -- Filter before grouping
GROUP BY department;
```

### 4.2 HAVING Clause
Filters groups **after** grouping and aggregation.

**Example:** Find departments with average salary > $75,000
```sql
SELECT 
    department,
    AVG(salary) AS avg_salary,
    COUNT(*) AS employee_count
FROM employees
GROUP BY department
HAVING AVG(salary) > 75000;  -- Filter after grouping
```

### 4.3 WHERE and HAVING Together
**Interview Question:** "Find departments with more than 5 employees hired after 2020, where the average salary exceeds $70,000"

```sql
SELECT 
    department,
    COUNT(*) AS employee_count,
    AVG(salary) AS avg_salary
FROM employees
WHERE hire_date > '2020-01-01'      -- Filter rows first
GROUP BY department
HAVING COUNT(*) > 5                 -- Filter groups
   AND AVG(salary) > 70000;
```

### 4.4 Performance Considerations
```sql
-- GOOD: Filter early with WHERE
SELECT department, AVG(salary)
FROM employees
WHERE status = 'ACTIVE'  -- Reduces rows before grouping
GROUP BY department
HAVING COUNT(*) > 10;

-- LESS EFFICIENT: Could filter earlier
SELECT department, AVG(salary)
FROM employees
GROUP BY department
HAVING COUNT(*) > 10
   AND status = 'ACTIVE';  -- This won't work as intended!
```

**Pro Tip:** Use WHERE to filter individual rows, HAVING to filter grouped results.

---

## 5. NULL Handling - Managing Missing Data

NULL handling is crucial in real-world data scenarios.

### 5.1 IS NULL vs IS NOT NULL
```sql
-- Find customers without phone numbers
SELECT customer_name
FROM customers
WHERE phone IS NULL;

-- Find customers with phone numbers
SELECT customer_name
FROM customers
WHERE phone IS NOT NULL;
```

### 5.2 COALESCE Function
Returns the first non-NULL value from a list.

**Example:** Display customer contact preference
```sql
SELECT 
    customer_name,
    COALESCE(mobile_phone, home_phone, work_phone, 'No phone') AS contact_number
FROM customers;
```

**Interview Question:** "Calculate total sales, treating NULL values as 0"
```sql
SELECT 
    product_id,
    SUM(COALESCE(quantity, 0) * COALESCE(unit_price, 0)) AS total_sales
FROM order_details
GROUP BY product_id;
```

### 5.3 NULLIF Function
Returns NULL if two expressions are equal.

**Example:** Avoid division by zero
```sql
SELECT 
    employee_name,
    total_sales,
    target_sales,
    ROUND(total_sales / NULLIF(target_sales, 0) * 100, 2) AS achievement_percentage
FROM sales_performance;
```

### 5.4 NULL in Aggregations
```sql
-- COUNT(*) includes NULLs, COUNT(column) excludes NULLs
SELECT 
    COUNT(*) AS total_rows,
    COUNT(phone) AS customers_with_phone,
    COUNT(*) - COUNT(phone) AS customers_without_phone
FROM customers;
```

### 5.5 NULL in JOINs
```sql
-- LEFT JOIN shows NULL for non-matching rows
SELECT 
    c.customer_name,
    COALESCE(o.order_count, 0) AS order_count
FROM customers c
LEFT JOIN (
    SELECT customer_id, COUNT(*) AS order_count
    FROM orders
    GROUP BY customer_id
) o ON c.customer_id = o.customer_id;
```

### 5.6 CASE WHEN for NULL Handling
```sql
SELECT 
    product_name,
    CASE 
        WHEN price IS NULL THEN 'Price not set'
        WHEN price = 0 THEN 'Free'
        WHEN price < 10 THEN 'Budget'
        ELSE 'Premium'
    END AS price_category
FROM products;
```

---

## 6. SQL Order of Operations - Understanding Query Execution

Understanding the logical order of SQL operations is crucial for writing efficient queries.

### 6.1 Logical Order of Operations

1. **FROM** - Identify source tables
2. **JOIN** - Combine tables
3. **WHERE** - Filter rows
4. **GROUP BY** - Group rows
5. **HAVING** - Filter groups
6. **SELECT** - Choose columns
7. **DISTINCT** - Remove duplicates
8. **ORDER BY** - Sort results
9. **LIMIT/OFFSET** - Limit results

### 6.2 Why Order Matters

**Example:** Understanding alias availability
```sql
-- This WORKS - alias available in ORDER BY
SELECT 
    customer_name,
    SUM(total_amount) AS total_spent
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
GROUP BY customer_name
ORDER BY total_spent DESC;  -- Alias available here

-- This FAILS - alias not available in WHERE
SELECT 
    customer_name,
    SUM(total_amount) AS total_spent
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE total_spent > 1000    -- ERROR: total_spent not yet calculated
GROUP BY customer_name;
```

### 6.3 Correct Approach
```sql
-- Use HAVING instead of WHERE for aggregated conditions
SELECT 
    customer_name,
    SUM(total_amount) AS total_spent
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
GROUP BY customer_name
HAVING SUM(total_amount) > 1000  -- Correct: filter after aggregation
ORDER BY total_spent DESC;
```

### 6.4 Performance Implications
```sql
-- GOOD: Filter early
SELECT 
    department,
    AVG(salary) AS avg_salary
FROM employees
WHERE status = 'ACTIVE'     -- Filter before grouping
  AND hire_date > '2020-01-01'
GROUP BY department
HAVING COUNT(*) > 5;

-- LESS EFFICIENT: Late filtering
SELECT 
    department,
    AVG(salary) AS avg_salary
FROM employees
GROUP BY department
HAVING COUNT(*) > 5
  AND MIN(status) = 'ACTIVE'  -- Could have filtered earlier
  AND MIN(hire_date) > '2020-01-01';
```

---

## 7. Window Functions - Advanced Analytics

Window functions perform calculations across rows related to the current row without collapsing the result set.

### 7.1 Ranking Functions

| Function | Description | Tie Handling |
|----------|-------------|--------------|
| `ROW_NUMBER()` | Assigns unique sequential numbers | Arbitrarily breaks ties |
| `RANK()` | Assigns rank with gaps for ties | Same rank for ties, gaps follow |
| `DENSE_RANK()` | Assigns rank without gaps | Same rank for ties, no gaps |
| `NTILE(n)` | Divides rows into n buckets | Distributes evenly |

**Example:** Rank employees by salary within each department
```sql
SELECT 
    employee_name,
    department,
    salary,
    ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS row_num,
    RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS rank_num,
    DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dense_rank
FROM employees;
```

### 7.2 Analytic Functions

**Example:** Calculate running totals and moving averages
```sql
SELECT 
    order_date,
    daily_sales,
    SUM(daily_sales) OVER (ORDER BY order_date) AS running_total,
    AVG(daily_sales) OVER (
        ORDER BY order_date 
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS seven_day_avg
FROM daily_sales_summary
ORDER BY order_date;
```

### 7.3 LAG and LEAD Functions

**Interview Question:** "Calculate month-over-month growth"
```sql
SELECT 
    sales_month,
    monthly_revenue,
    LAG(monthly_revenue, 1) OVER (ORDER BY sales_month) AS prev_month_revenue,
    ROUND(
        (monthly_revenue - LAG(monthly_revenue, 1) OVER (ORDER BY sales_month)) 
        / LAG(monthly_revenue, 1) OVER (ORDER BY sales_month) * 100, 2
    ) AS growth_percentage
FROM monthly_sales;
```

### 7.4 FIRST_VALUE and LAST_VALUE

**Example:** Compare each employee's salary to department min/max
```sql
SELECT 
    employee_name,
    department,
    salary,
    FIRST_VALUE(salary) OVER (
        PARTITION BY department 
        ORDER BY salary DESC
        ROWS UNBOUNDED PRECEDING
    ) AS dept_max_salary,
    LAST_VALUE(salary) OVER (
        PARTITION BY department 
        ORDER BY salary DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS dept_min_salary
FROM employees;
```

---

## 8. Datetime Functions - Time Manipulation

### 8.1 Current Date/Time Functions

| Function | Description | Example |
|----------|-------------|---------|
| `NOW()` / `CURRENT_TIMESTAMP` | Current date and time | `2024-05-19 16:45:23` |
| `CURDATE()` / `CURRENT_DATE` | Current date only | `2024-05-19` |
| `CURTIME()` / `CURRENT_TIME` | Current time only | `16:45:23` |

**Example:** Record user login
```sql
INSERT INTO user_sessions (user_id, login_time, session_date)
VALUES (12345, NOW(), CURDATE());
```

### 8.2 Date Extraction Functions

| Function | Description | Example Result |
|----------|-------------|----------------|
| `YEAR(date)` | Extract year | `2024` |
| `MONTH(date)` | Extract month (1-12) | `5` |
| `DAY(date)` | Extract day (1-31) | `19` |
| `DAYOFWEEK(date)` | Day of week (1=Sunday) | `1` |
| `DAYNAME(date)` | Day name | `'Sunday'` |
| `MONTHNAME(date)` | Month name | `'May'` |

**Interview Question:** "Find total sales by month for 2024"
```sql
SELECT 
    MONTHNAME(order_date) AS month_name,
    MONTH(order_date) AS month_num,
    SUM(total_amount) AS monthly_sales
FROM orders
WHERE YEAR(order_date) = 2024
GROUP BY MONTH(order_date), MONTHNAME(order_date)
ORDER BY month_num;
```

### 8.3 Date Arithmetic

| Function | Description | Example |
|----------|-------------|---------|
| `DATE_ADD(date, INTERVAL expr unit)` | Add time interval | `DATE_ADD('2024-05-19', INTERVAL 30 DAY)` |
| `DATE_SUB(date, INTERVAL expr unit)` | Subtract time interval | `DATE_SUB(NOW(), INTERVAL 1 YEAR)` |
| `DATEDIFF(date1, date2)` | Difference in days | `DATEDIFF('2024-12-31', '2024-01-01')` |
| `TIMESTAMPDIFF(unit, start, end)` | Difference in specified unit | `TIMESTAMPDIFF(MONTH, '2024-01-01', '2024-06-01')` |

**Example:** Find customers who haven't ordered in the last 6 months
```sql
SELECT 
    c.customer_name,
    MAX(o.order_date) AS last_order_date,
    DATEDIFF(CURDATE(), MAX(o.order_date)) AS days_since_last_order
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.customer_name
HAVING MAX(o.order_date) < DATE_SUB(CURDATE(), INTERVAL 6 MONTH)
   OR MAX(o.order_date) IS NULL;
```

### 8.4 Date Formatting

**Example:** Format dates for reports
```sql
SELECT 
    order_id,
    DATE_FORMAT(order_date, '%W, %M %d, %Y') AS formatted_date,
    DATE_FORMAT(order_date, '%Y-%m') AS year_month
FROM orders;
-- Output: 'Sunday, May 19, 2024', '2024-05'
```

---

## 9. String Functions - Text Processing

String manipulation is essential for data cleaning and formatting.

### 9.1 Basic String Functions

| Function | Description | Example | Result |
|----------|-------------|---------|--------|
| `CONCAT(s1, s2, ...)` | Join strings | `CONCAT('Hello', ' ', 'World')` | `'Hello World'` |
| `CONCAT_WS(sep, s1, s2, ...)` | Join with separator | `CONCAT_WS(', ', 'John', 'Doe')` | `'John, Doe'` |
| `LENGTH(str)` | String length in bytes | `LENGTH('hello')` | `5` |
| `CHAR_LENGTH(str)` | String length in characters | `CHAR_LENGTH('hello')` | `5` |
| `UPPER(str)` / `LOWER(str)` | Change case | `UPPER('Hello')` | `'HELLO'` |

### 9.2 String Manipulation

| Function | Description | Example | Result |
|----------|-------------|---------|--------|
| `SUBSTRING(str, pos, len)` | Extract substring | `SUBSTRING('database', 5, 4)` | `'base'` |
| `LEFT(str, len)` / `RIGHT(str, len)` | Extract from left/right | `LEFT('database', 4)` | `'data'` |
| `TRIM(str)` | Remove whitespace | `TRIM('  hello  ')` | `'hello'` |
| `LTRIM(str)` / `RTRIM(str)` | Remove left/right whitespace | `LTRIM('  hello')` | `'hello'` |
| `REPLACE(str, from, to)` | Replace substring | `REPLACE('a-b-c', '-', ' ')` | `'a b c'` |

**Interview Question:** "Clean and standardize customer data"
```sql
SELECT 
    customer_id,
    TRIM(UPPER(customer_name)) AS clean_name,
    LOWER(TRIM(email)) AS clean_email,
    REPLACE(REPLACE(phone, '-', ''), ' ', '') AS clean_phone
FROM customers_raw;
```

### 9.3 Pattern Matching

**Example:** Find customers with Gmail addresses
```sql
SELECT customer_name, email
FROM customers
WHERE email LIKE '%@gmail.com'
   OR email REGEXP '@gmail\\.com$';
```

### 9.4 String Aggregation

**Example:** List all products per category
```sql
SELECT 
    category,
    GROUP_CONCAT(
        product_name 
        ORDER BY product_name 
        SEPARATOR ', '
    ) AS products
FROM products
GROUP BY category;
```

---

## 10. Aggregation & Grouping - Data Summarization

### 10.1 Core Aggregate Functions

| Function | Description | NULL Handling |
|----------|-------------|---------------|
| `COUNT(*)` | Count all rows | Includes NULLs |
| `COUNT(column)` | Count non-NULL values | Excludes NULLs |
| `SUM(column)` | Calculate sum | Excludes NULLs |
| `AVG(column)` | Calculate average | Excludes NULLs |
| `MIN(column)` / `MAX(column)` | Find min/max | Excludes NULLs |

**Interview Question:** "Calculate sales statistics by product category"
```sql
SELECT 
    category,
    COUNT(*) AS total_products,
    COUNT(price) AS products_with_price,
    ROUND(AVG(price), 2) AS avg_price,
    MIN(price) AS min_price,
    MAX(price) AS max_price,
    SUM(CASE WHEN price > 100 THEN 1 ELSE 0 END) AS premium_products
FROM products
GROUP BY category
ORDER BY avg_price DESC;
```

### 10.2 GROUP BY with Multiple Columns

**Example:** Sales analysis by year and quarter
```sql
SELECT 
    YEAR(order_date) AS sales_year,
    QUARTER(order_date) AS sales_quarter,
    COUNT(*) AS order_count,
    SUM(total_amount) AS total_sales,
    AVG(total_amount) AS avg_order_value
FROM orders
GROUP BY YEAR(order_date), QUARTER(order_date)
ORDER BY sales_year, sales_quarter;
```

### 10.3 Conditional Aggregation

**Example:** Pivot-like analysis using CASE
```sql
SELECT 
    customer_id,
    SUM(CASE WHEN YEAR(order_date) = 2023 THEN total_amount ELSE 0 END) AS sales_2023,
    SUM(CASE WHEN YEAR(order_date) = 2024 THEN total_amount ELSE 0 END) AS sales_2024,
    COUNT(CASE WHEN status = 'completed' THEN 1 END) AS completed_orders,
    COUNT(CASE WHEN status = 'cancelled' THEN 1 END) AS cancelled_orders
FROM orders
GROUP BY customer_id;
```

---

## 11. Advanced Concepts - Professional Techniques

### 11.1 CASE Statements
Add conditional logic to queries.

**Example:** Categorize customers by purchase behavior
```sql
SELECT 
    customer_name,
    total_orders,
    total_spent,
    CASE 
        WHEN total_orders >= 10 AND total_spent >= 1000 THEN 'VIP'
        WHEN total_orders >= 5 OR total_spent >= 500 THEN 'Regular'
        WHEN total_orders >= 1 THEN 'Occasional'
        ELSE 'Prospect'
    END AS customer_tier
FROM customer_summary;
```

### 11.2 Pagination with LIMIT and OFFSET

**Example:** Implement pagination for large result sets
```sql
-- Page 3 of results (20 items per page)
SELECT 
    product_name,
    price,
    category
FROM products
ORDER BY created_date DESC
LIMIT 20 OFFSET 40;  -- Skip 40 (pages 1-2), take 20 (page 3)
```

### 11.3 UNION vs UNION ALL Performance

**Example:** Combining historical data
```sql
-- Use UNION ALL when you know there are no duplicates (faster)
SELECT customer_id, order_date, 'current' AS table_source
FROM orders_current
UNION ALL
SELECT customer_id, order_date, 'archive' AS table_source
FROM orders_archive
ORDER BY order_date DESC;
```

### 11.4 Handling Data Type Conversions

**Example:** Safe type casting
```sql
SELECT 
    product_id,
    CAST(price AS DECIMAL(10,2)) AS formatted_price,
    CAST(created_date AS DATE) AS creation_date
FROM products_staging
WHERE price REGEXP '^[0-9]+\\.?[0-9]*$';  -- Validate before casting
```

### 11.5 Advanced Pattern Matching

**Example:** Find products with specific naming patterns
```sql
SELECT product_name
FROM products
WHERE product_name REGEXP '^[A-Z]{2,3}-[0-9]{3,4}$'  -- Format: AB-123 or ABC-1234
   OR product_name LIKE 'Pro_%'                        -- Starts with 'Pro_'
   OR product_name LIKE '%_v[0-9]'                     -- Ends with version number
ORDER BY product_name;
```

---

## Common Interview Questions & Solutions

### Question 1: Second Highest Salary
"Find the second highest salary in the employees table."

```sql
-- Method 1: Using LIMIT with subquery
SELECT MAX(salary) AS second_highest
FROM employees
WHERE salary < (SELECT MAX(salary) FROM employees);

-- Method 2: Using window functions
SELECT salary AS second_highest
FROM (
    SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS rank_num
    FROM employees
) ranked
WHERE rank_num = 2
LIMIT 1;
```

### Question 2: Duplicate Records
"Find all duplicate email addresses in the customers table."

```sql
SELECT 
    email,
    COUNT(*) AS duplicate_count
FROM customers
GROUP BY email
HAVING COUNT(*) > 1
ORDER BY duplicate_count DESC;
```

### Question 3: Consecutive Dates
"Find users who logged in on consecutive days."

```sql
WITH user_logins AS (
    SELECT 
        user_id,
        login_date,
        LAG(login_date, 1) OVER (PARTITION BY user_id ORDER BY login_date) AS prev_login
    FROM user_activity
)
SELECT DISTINCT user_id
FROM user_logins
WHERE DATEDIFF(login_date, prev_login) = 1;
```

### Question 4: Top N per Group
"Find the top 3 highest-paid employees in each department."

```sql
WITH ranked_employees AS (
    SELECT 
        employee_name,
        department,
        salary,
        ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS rank_num
    FROM employees
)
SELECT 
    department,
    employee_name,
    salary
FROM ranked_employees
WHERE rank_num <= 3
ORDER BY department, rank_num;
```

### Question 5: Running Totals
"Calculate running totals of sales by date."

```sql
SELECT 
    sale_date,
    daily_sales,
    SUM(daily_sales) OVER (ORDER BY sale_date ROWS UNBOUNDED PRECEDING) AS running_total
FROM (
    SELECT 
        DATE(order_date) AS sale_date,
        SUM(total_amount) AS daily_sales
    FROM orders
    GROUP BY DATE(order_date)
) daily_summary
ORDER BY sale_date;
```

### Question 6: Self-Join for Comparisons
"Find employees who earn more than their department's average salary."

```sql
SELECT 
    e.employee_name,
    e.department,
    e.salary,
    dept_avg.avg_salary
FROM employees e
INNER JOIN (
    SELECT 
        department,
        AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department
) dept_avg ON e.department = dept_avg.department
WHERE e.salary > dept_avg.avg_salary
ORDER BY e.department, e.salary DESC;
```

### Question 7: Complex Date Ranges
"Find customers who made purchases in both Q1 and Q4 of 2024."

```sql
SELECT c.customer_name
FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o1
    WHERE o1.customer_id = c.customer_id
      AND YEAR(o1.order_date) = 2024
      AND QUARTER(o1.order_date) = 1
)
AND EXISTS (
    SELECT 1 FROM orders o2
    WHERE o2.customer_id = c.customer_id
      AND YEAR(o2.order_date) = 2024
      AND QUARTER(o2.order_date) = 4
);
```

### Question 8: Hierarchical Data
"Display an organization chart showing employee-manager relationships."

```sql
WITH RECURSIVE org_chart AS (
    -- Base case: Top-level executives (no manager)
    SELECT 
        employee_id,
        employee_name,
        manager_id,
        0 AS level,
        CAST(employee_name AS CHAR(1000)) AS hierarchy_path
    FROM employees
    WHERE manager_id IS NULL
    
    UNION ALL
    
    -- Recursive case: Employees with managers
    SELECT 
        e.employee_id,
        e.employee_name,
        e.manager_id,
        oc.level + 1,
        CONCAT(oc.hierarchy_path, ' > ', e.employee_name)
    FROM employees e
    INNER JOIN org_chart oc ON e.manager_id = oc.employee_id
)
SELECT 
    CONCAT(REPEAT('  ', level), employee_name) AS org_structure,
    level,
    hierarchy_path
FROM org_chart
ORDER BY hierarchy_path;
```

### Question 9: Advanced Aggregation
"Calculate customer lifetime value (CLV) and rank customers."

```sql
WITH customer_metrics AS (
    SELECT 
        c.customer_id,
        c.customer_name,
        c.signup_date,
        COUNT(o.order_id) AS total_orders,
        COALESCE(SUM(o.total_amount), 0) AS total_spent,
        COALESCE(AVG(o.total_amount), 0) AS avg_order_value,
        DATEDIFF(CURDATE(), c.signup_date) AS days_as_customer,
        CASE 
            WHEN COUNT(o.order_id) = 0 THEN 0
            ELSE ROUND(SUM(o.total_amount) / NULLIF(DATEDIFF(CURDATE(), c.signup_date), 0) * 365, 2)
        END AS annual_value
    FROM customers c
    LEFT JOIN orders o ON c.customer_id = o.customer_id
    GROUP BY c.customer_id, c.customer_name, c.signup_date
)
SELECT 
    customer_name,
    total_orders,
    total_spent,
    avg_order_value,
    annual_value,
    NTILE(5) OVER (ORDER BY annual_value DESC) AS value_quintile,
    CASE 
        WHEN annual_value >= 1000 THEN 'High Value'
        WHEN annual_value >= 500 THEN 'Medium Value'
        WHEN annual_value > 0 THEN 'Low Value'
        ELSE 'No Value'
    END AS customer_segment
FROM customer_metrics
ORDER BY annual_value DESC;
```

### Question 10: Complex Time-Series Analysis
"Find products with declining sales over the last 3 months."

```sql
WITH monthly_sales AS (
    SELECT 
        p.product_id,
        p.product_name,
        DATE_FORMAT(o.order_date, '%Y-%m') AS sales_month,
        SUM(od.quantity * od.unit_price) AS monthly_revenue
    FROM products p
    INNER JOIN order_details od ON p.product_id = od.product_id
    INNER JOIN orders o ON od.order_id = o.order_id
    WHERE o.order_date >= DATE_SUB(CURDATE(), INTERVAL 3 MONTH)
    GROUP BY p.product_id, p.product_name, DATE_FORMAT(o.order_date, '%Y-%m')
),
sales_trends AS (
    SELECT 
        product_id,
        product_name,
        sales_month,
        monthly_revenue,
        LAG(monthly_revenue, 1) OVER (PARTITION BY product_id ORDER BY sales_month) AS prev_month,
        LAG(monthly_revenue, 2) OVER (PARTITION BY product_id ORDER BY sales_month) AS two_months_ago
    FROM monthly_sales
)
SELECT 
    product_name,
    sales_month,
    monthly_revenue,
    prev_month,
    two_months_ago,
    ROUND(
        CASE 
            WHEN prev_month > 0 THEN (monthly_revenue - prev_month) / prev_month * 100
            ELSE NULL
        END, 2
    ) AS month_over_month_change
FROM sales_trends
WHERE monthly_revenue < prev_month 
  AND prev_month < two_months_ago
  AND prev_month IS NOT NULL 
  AND two_months_ago IS NOT NULL
ORDER BY product_name, sales_month;
```

---

## Performance Tips & Best Practices

### 1. Indexing Strategy
```sql
-- Good: Use indexes for WHERE clauses
CREATE INDEX idx_customer_email ON customers(email);
CREATE INDEX idx_order_date ON orders(order_date);
CREATE INDEX idx_product_category ON products(category);

-- Composite indexes for multi-column queries
CREATE INDEX idx_orders_customer_date ON orders(customer_id, order_date);
```

### 2. Query Optimization
```sql
-- GOOD: Filter early
SELECT c.customer_name, o.order_date
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_date >= '2024-01-01'  -- Filter reduces join size
  AND c.status = 'ACTIVE';

-- AVOID: Late filtering
SELECT c.customer_name, o.order_date
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_date >= '2024-01-01'
  AND c.status = 'ACTIVE';  -- Same logic, but demonstrates the principle
```

### 3. EXISTS vs IN Performance
```sql
-- GOOD: Use EXISTS for better performance with large datasets
SELECT customer_name
FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o 
    WHERE o.customer_id = c.customer_id 
      AND o.order_date >= '2024-01-01'
);

-- CAN BE SLOWER: IN with subquery
SELECT customer_name
FROM customers c
WHERE customer_id IN (
    SELECT customer_id FROM orders 
    WHERE order_date >= '2024-01-01'
);
```

### 4. Avoid Functions in WHERE Clauses
```sql
-- GOOD: Uses index
SELECT * FROM orders 
WHERE order_date >= '2024-01-01' 
  AND order_date < '2024-02-01';

-- AVOID: Prevents index usage
SELECT * FROM orders 
WHERE YEAR(order_date) = 2024 
  AND MONTH(order_date) = 1;
```

### 5. Limit Result Sets
```sql
-- Always use LIMIT for large queries in development
SELECT * FROM large_table 
ORDER BY created_date DESC 
LIMIT 100;

-- Use pagination for user interfaces
SELECT * FROM products 
ORDER BY product_name 
LIMIT 20 OFFSET 40;  -- Page 3, 20 items per page
```

---

## Common Mistakes to Avoid

### 1. NULL Comparisons
```sql
-- WRONG: This won't work as expected
SELECT * FROM customers WHERE phone = NULL;

-- CORRECT: Use IS NULL
SELECT * FROM customers WHERE phone IS NULL;
```

### 2. String Comparisons
```sql
-- WRONG: Case-sensitive comparison
SELECT * FROM products WHERE product_name = 'laptop';

-- BETTER: Case-insensitive
SELECT * FROM products WHERE LOWER(product_name) = 'laptop';
```

### 3. Date Comparisons
```sql
-- WRONG: Might miss records due to time component
SELECT * FROM orders WHERE order_date = '2024-05-19';

-- BETTER: Use date ranges
SELECT * FROM orders 
WHERE order_date >= '2024-05-19' 
  AND order_date < '2024-05-20';
```

### 4. Aggregation with NULL
```sql
-- Be aware: COUNT(*) vs COUNT(column)
SELECT 
    COUNT(*) AS total_customers,          -- Includes NULLs
    COUNT(phone) AS customers_with_phone  -- Excludes NULLs
FROM customers;
```

### 5. JOIN Conditions
```sql
-- WRONG: Missing JOIN condition creates Cartesian product
SELECT c.customer_name, o.order_id
FROM customers c, orders o;  -- Dangerous!

-- CORRECT: Always specify JOIN conditions
SELECT c.customer_name, o.order_id
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id;
```

---

## Conclusion

This guide covers the essential SQL concepts and techniques commonly tested in technical interviews. Remember these key points:

1. **Master JOINs** - They're the foundation of relational queries
2. **Understand execution order** - Know when WHERE vs HAVING applies
3. **Practice window functions** - They're increasingly common in interviews
4. **Handle NULLs properly** - Real data is messy
5. **Think about performance** - Write efficient queries from the start
6. **Use CTEs for clarity** - Make complex queries readable
7. **Test edge cases** - Consider empty results, NULLs, and duplicates

Practice these concepts with real datasets, and you'll be well-prepared for any SQL coding interview!