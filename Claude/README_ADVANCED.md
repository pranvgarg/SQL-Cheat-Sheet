# SQL Cheat Sheet - Advanced (Claude)

## Table of Contents
1. [Window Functions](#window-functions)
2. [Common Table Expressions](#common-table-expressions)
3. [Advanced Joins](#advanced-joins)
4. [Performance Optimization](#performance-optimization)
5. [Stored Procedures and Functions](#stored-procedures-and-functions)
6. [Advanced Data Types](#advanced-data-types)

## Window Functions
```sql
-- ROW_NUMBER()
SELECT 
    employee_id,
    name,
    salary,
    department,
    ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) as rank_in_dept
FROM employees;

-- RANK() and DENSE_RANK()
SELECT 
    product_id,
    name,
    price,
    RANK() OVER (ORDER BY price DESC) as price_rank,
    DENSE_RANK() OVER (ORDER BY price DESC) as dense_price_rank
FROM products;

-- Running totals
SELECT 
    order_date,
    amount,
    SUM(amount) OVER (ORDER BY order_date) as running_total
FROM orders;
```

## Common Table Expressions (CTEs)
```sql
-- Basic CTE
WITH dept_avg_salary AS (
    SELECT 
        department,
        AVG(salary) as avg_salary
    FROM employees
    GROUP BY department
)
SELECT 
    e.name,
    e.salary,
    e.department,
    das.avg_salary as dept_avg
FROM employees e
JOIN dept_avg_salary das ON e.department = das.department
WHERE e.salary > das.avg_salary;

-- Recursive CTE (for hierarchical data)
WITH RECURSIVE org_chart AS (
    -- Base case
    SELECT 
        id,
        name,
        manager_id,
        1 as level
    FROM employees
    WHERE manager_id IS NULL
    
    UNION ALL
    
    -- Recursive case
    SELECT 
        e.id,
        e.name,
        e.manager_id,
        oc.level + 1
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.id
)
SELECT * FROM org_chart ORDER BY level, name;
```

## Advanced Joins
```sql
-- Self join
SELECT 
    e1.name as employee,
    e2.name as manager
FROM employees e1
LEFT JOIN employees e2 ON e1.manager_id = e2.id;

-- CROSS APPLY (SQL Server)
SELECT 
    d.department_name,
    top_emp.name,
    top_emp.salary
FROM departments d
CROSS APPLY (
    SELECT TOP 3 name, salary
    FROM employees e
    WHERE e.department_id = d.id
    ORDER BY salary DESC
) AS top_emp;
```

## Performance Optimization
```sql
-- Indexing
CREATE INDEX idx_employee_department ON employees(department_id);
CREATE INDEX idx_order_date ON orders(order_date);

-- Query hints (SQL Server)
SELECT * FROM products WITH (NOLOCK) WHERE category = 'Electronics';

-- Analyze query execution plan
EXPLAIN ANALYZE 
SELECT * FROM orders 
WHERE order_date > '2023-01-01' 
ORDER BY total_amount DESC;

-- Partitioning (PostgreSQL)
CREATE TABLE sales (
    id SERIAL,
    sale_date DATE,
    amount DECIMAL(10,2),
    region VARCHAR(50)
) PARTITION BY RANGE (sale_date);

-- Create monthly partitions
CREATE TABLE sales_y2023m01 PARTITION OF sales
    FOR VALUES FROM ('2023-01-01') TO ('2023-02-01');
```

## Stored Procedures and Functions
```sql
-- Stored procedure
DELIMITER //
CREATE PROCEDURE GetEmployeeCount(IN dept_name VARCHAR(100), OUT emp_count INT)
BEGIN
    SELECT COUNT(*) INTO emp_count
    FROM employees
    WHERE department = dept_name;
END //
DELIMITER ;

-- Call stored procedure
CALL GetEmployeeCount('Engineering', @count);
SELECT @count;

-- Function
DELIMITER //
CREATE FUNCTION GetDepartmentBudget(dept_id INT) 
RETURNS DECIMAL(10,2)
DETERMINISTIC
BEGIN
    DECLARE total_budget DECIMAL(10,2);
    
    SELECT SUM(salary) INTO total_budget
    FROM employees
    WHERE department_id = dept_id;
    
    RETURN total_budget;
END //
DELIMITER ;

-- Use function
SELECT name, GetDepartmentBudget(id) as budget FROM departments;
```

## Advanced Data Types
```sql
-- JSON (PostgreSQL/MySQL 5.7+)
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    attributes JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insert JSON data
INSERT INTO products (name, attributes) 
VALUES ('Smartphone', '{"brand": "Samsung", "storage": "128GB", "color": "black"}');

-- Query JSON data
SELECT 
    name,
    attributes->>'brand' as brand,
    attributes->'storage' as storage
FROM products
WHERE attributes->>'color' = 'black';

-- Full-text search
CREATE TABLE articles (
    id SERIAL PRIMARY KEY,
    title VARCHAR(200),
    content TEXT,
    FULLTEXT(title, content)
);

-- Search with relevance
SELECT 
    id,
    title,
    MATCH(title, content) AGAINST('database performance') as relevance
FROM articles
WHERE MATCH(title, content) AGAINST('database performance')
ORDER BY relevance DESC;
```

This advanced cheat sheet covers complex SQL concepts and techniques for experienced database developers and administrators.


# The Ultimate SQL Interview Preparation Guide

A comprehensive guide covering essential SQL concepts with real-world examples, advanced techniques, and interview-focused solutions.

---

## Table of Contents

### **Foundation Level**
1. [JOINs - The Foundation of Relational Queries](#1-joins---the-foundation-of-relational-queries)
2. [Basic Filtering - WHERE Clauses](#2-basic-filtering---where-clauses)
3. [Aggregation & Grouping](#3-aggregation--grouping)
4. [NULL Handling](#4-null-handling---managing-missing-data)

### **Intermediate Level**
5. [Subqueries & CTEs](#5-subqueries--ctes---advanced-query-composition)
6. [Set Operations](#6-set-operations---combining-result-sets)
7. [Advanced Filtering - WHERE vs HAVING](#7-advanced-filtering---where-vs-having)
8. [String Functions](#8-string-functions---text-processing)
9. [Date & Time Functions](#9-date--time-functions)

### **Advanced Level**
10. [Window Functions](#10-window-functions---advanced-analytics)
11. [Advanced SQL Techniques](#11-advanced-sql-techniques)
12. [Query Optimization](#12-query-optimization--performance)
13. [SQL Execution Order](#13-sql-execution-order)

### **Interview Preparation**
14. [Common Interview Questions](#14-common-interview-questions--solutions)
15. [Advanced Interview Scenarios](#15-advanced-interview-scenarios)
16. [Best Practices & Performance Tips](#16-best-practices--performance-tips)

---

## Sample Database Schema

For consistency across all examples, we'll use this e-commerce database schema:

```sql
-- Customers table
CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    customer_name VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE,
    phone VARCHAR(20),
    city VARCHAR(50),
    country VARCHAR(50),
    signup_date DATE,
    status ENUM('ACTIVE', 'INACTIVE') DEFAULT 'ACTIVE'
);

-- Products table
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(100) NOT NULL,
    category VARCHAR(50),
    price DECIMAL(10,2),
    stock_quantity INT,
    created_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Orders table
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    order_date DATE,
    total_amount DECIMAL(10,2),
    status ENUM('PENDING', 'COMPLETED', 'CANCELLED'),
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

-- Order Details table
CREATE TABLE order_details (
    order_detail_id INT PRIMARY KEY,
    order_id INT,
    product_id INT,
    quantity INT,
    unit_price DECIMAL(10,2),
    FOREIGN KEY (order_id) REFERENCES orders(order_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);

-- Employees table
CREATE TABLE employees (
    employee_id INT PRIMARY KEY,
    employee_name VARCHAR(100),
    department VARCHAR(50),
    salary DECIMAL(10,2),
    manager_id INT,
    hire_date DATE,
    status ENUM('ACTIVE', 'INACTIVE') DEFAULT 'ACTIVE',
    FOREIGN KEY (manager_id) REFERENCES employees(employee_id)
);
```

---

## 1. JOINs - The Foundation of Relational Queries

JOINs are the most critical concept in SQL interviews. Understanding different types and their use cases is essential.

### 1.1 INNER JOIN
Returns only matching records from both tables.

**Basic Syntax:**
```sql
SELECT columns
FROM table1 t1
INNER JOIN table2 t2 ON t1.common_column = t2.common_column;
```

**Example 1: Orders with Customer Information**
```sql
SELECT 
    o.order_id,
    o.order_date,
    c.customer_name,
    c.email,
    o.total_amount
FROM orders o
INNER JOIN customers c ON o.customer_id = c.customer_id
WHERE o.status = 'COMPLETED'
ORDER BY o.order_date DESC;
```

**Example 2: Products Sold (with quantities)**
```sql
SELECT 
    p.product_name,
    p.category,
    SUM(od.quantity) AS total_sold,
    SUM(od.quantity * od.unit_price) AS total_revenue
FROM products p
INNER JOIN order_details od ON p.product_id = od.product_id
INNER JOIN orders o ON od.order_id = o.order_id
WHERE o.status = 'COMPLETED'
GROUP BY p.product_id, p.product_name, p.category
ORDER BY total_revenue DESC;
```

### 1.2 LEFT JOIN (LEFT OUTER JOIN)
Returns all records from the left table and matching records from the right table.

**Example 1: All Customers with Their Order Count**
```sql
SELECT 
    c.customer_name,
    c.email,
    c.signup_date,
    COUNT(o.order_id) AS total_orders,
    COALESCE(SUM(o.total_amount), 0) AS total_spent
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.customer_name, c.email, c.signup_date
ORDER BY total_spent DESC;
```

**Example 2: Find Customers Who Never Ordered**
```sql
SELECT 
    c.customer_name,
    c.email,
    c.signup_date,
    DATEDIFF(CURDATE(), c.signup_date) AS days_since_signup
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE o.customer_id IS NULL
ORDER BY c.signup_date;
```

### 1.3 RIGHT JOIN (RIGHT OUTER JOIN)
Returns all records from the right table and matching records from the left table.

**Example: All Products with Their Sales Data**
```sql
SELECT 
    p.product_name,
    p.category,
    p.price,
    COALESCE(SUM(od.quantity), 0) AS units_sold,
    COALESCE(SUM(od.quantity * od.unit_price), 0) AS revenue
FROM order_details od
RIGHT JOIN products p ON od.product_id = p.product_id
GROUP BY p.product_id, p.product_name, p.category, p.price
ORDER BY revenue DESC;
```

### 1.4 FULL OUTER JOIN
Returns all records when there's a match in either table.

**Example: Complete Customer and Order Overview**
```sql
-- Note: MySQL doesn't support FULL OUTER JOIN directly
-- Use UNION of LEFT and RIGHT JOINs
SELECT 
    c.customer_name,
    o.order_id,
    o.order_date,
    o.total_amount
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id

UNION

SELECT 
    c.customer_name,
    o.order_id,
    o.order_date,
    o.total_amount
FROM customers c
RIGHT JOIN orders o ON c.customer_id = o.customer_id
ORDER BY customer_name, order_date;
```

### 1.5 SELF JOIN
Joining a table with itself for hierarchical relationships.

**Example 1: Employee-Manager Relationships**
```sql
SELECT 
    e.employee_name AS employee,
    e.department,
    e.salary,
    m.employee_name AS manager,
    m.salary AS manager_salary
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.employee_id
ORDER BY e.department, e.employee_name;
```

**Example 2: Find Employees Earning More Than Their Managers**
```sql
SELECT 
    e.employee_name,
    e.salary AS employee_salary,
    m.employee_name AS manager,
    m.salary AS manager_salary,
    (e.salary - m.salary) AS salary_difference
FROM employees e
INNER JOIN employees m ON e.manager_id = m.employee_id
WHERE e.salary > m.salary
ORDER BY salary_difference DESC;
```

### 1.6 CROSS JOIN
Cartesian product of two tables - use with extreme caution!

**Example: Product-Category Matrix for Analysis**
```sql
SELECT 
    p.product_name,
    c.category_name,
    CASE 
        WHEN p.category = c.category_name THEN 'MATCH'
        ELSE 'NO MATCH'
    END AS category_match
FROM products p
CROSS JOIN (SELECT DISTINCT category AS category_name FROM products) c
WHERE p.product_id <= 5  -- Limit for demonstration
ORDER BY p.product_name, c.category_name;
```

---

## 2. Basic Filtering - WHERE Clauses

Effective filtering is fundamental to efficient querying.

### 2.1 Comparison Operators

**Example: Product Price Filtering**
```sql
SELECT 
    product_name,
    category,
    price,
    stock_quantity
FROM products
WHERE price BETWEEN 10.00 AND 100.00
  AND stock_quantity > 0
  AND category IN ('Electronics', 'Books', 'Clothing')
ORDER BY price DESC;
```

### 2.2 Pattern Matching with LIKE

**Example: Customer Search**
```sql
SELECT 
    customer_name,
    email,
    city
FROM customers
WHERE customer_name LIKE 'J%'           -- Starts with 'J'
   OR email LIKE '%gmail.com'           -- Gmail users
   OR city LIKE '%New%'                 -- Cities containing 'New'
ORDER BY customer_name;
```

### 2.3 Complex Conditions

**Example: Order Analysis with Multiple Conditions**
```sql
SELECT 
    o.order_id,
    c.customer_name,
    o.order_date,
    o.total_amount,
    o.status
FROM orders o
INNER JOIN customers c ON o.customer_id = c.customer_id
WHERE (o.total_amount > 500 OR c.city = 'New York')
  AND o.order_date >= DATE_SUB(CURDATE(), INTERVAL 30 DAY)
  AND o.status != 'CANCELLED'
ORDER BY o.total_amount DESC;
```

---

## 3. Aggregation & Grouping

Data summarization is crucial for business intelligence and reporting.

### 3.1 Basic Aggregate Functions

| Function | Description | NULL Handling |
|----------|-------------|---------------|
| `COUNT(*)` | Count all rows | Includes NULLs |
| `COUNT(column)` | Count non-NULL values | Excludes NULLs |
| `SUM(column)` | Calculate sum | Excludes NULLs |
| `AVG(column)` | Calculate average | Excludes NULLs |
| `MIN(column)` / `MAX(column)` | Find minimum/maximum | Excludes NULLs |

**Example: Sales Summary by Category**
```sql
SELECT 
    p.category,
    COUNT(*) AS total_products,
    COUNT(p.price) AS products_with_price,
    MIN(p.price) AS min_price,
    MAX(p.price) AS max_price,
    ROUND(AVG(p.price), 2) AS avg_price,
    SUM(CASE WHEN p.price > 100 THEN 1 ELSE 0 END) AS premium_count
FROM products p
GROUP BY p.category
HAVING COUNT(*) > 5
ORDER BY avg_price DESC;
```

### 3.2 GROUP BY with Multiple Columns

**Example: Monthly Sales Analysis**
```sql
SELECT 
    YEAR(o.order_date) AS sales_year,
    MONTH(o.order_date) AS sales_month,
    MONTHNAME(o.order_date) AS month_name,
    COUNT(o.order_id) AS order_count,
    COUNT(DISTINCT o.customer_id) AS unique_customers,
    SUM(o.total_amount) AS total_sales,
    ROUND(AVG(o.total_amount), 2) AS avg_order_value
FROM orders o
WHERE o.status = 'COMPLETED'
  AND o.order_date >= '2024-01-01'
GROUP BY YEAR(o.order_date), MONTH(o.order_date), MONTHNAME(o.order_date)
ORDER BY sales_year, sales_month;
```

### 3.3 Conditional Aggregation

**Example: Customer Behavior Analysis**
```sql
SELECT 
    c.customer_id,
    c.customer_name,
    COUNT(o.order_id) AS total_orders,
    SUM(CASE WHEN o.status = 'COMPLETED' THEN 1 ELSE 0 END) AS completed_orders,
    SUM(CASE WHEN o.status = 'CANCELLED' THEN 1 ELSE 0 END) AS cancelled_orders,
    SUM(CASE WHEN YEAR(o.order_date) = 2024 THEN o.total_amount ELSE 0 END) AS sales_2024,
    SUM(CASE WHEN YEAR(o.order_date) = 2023 THEN o.total_amount ELSE 0 END) AS sales_2023,
    ROUND(
        SUM(CASE WHEN o.status = 'COMPLETED' THEN o.total_amount ELSE 0 END) / 
        NULLIF(SUM(CASE WHEN o.status = 'COMPLETED' THEN 1 ELSE 0 END), 0), 2
    ) AS avg_completed_order_value
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.customer_name
HAVING COUNT(o.order_id) > 0
ORDER BY total_orders DESC;
```

---

## 4. NULL Handling - Managing Missing Data

Proper NULL handling is critical in real-world applications.

### 4.1 NULL Comparison Operators

```sql
-- Correct NULL comparisons
SELECT 
    customer_name,
    email,
    phone,
    CASE 
        WHEN phone IS NULL THEN 'No phone provided'
        WHEN phone = '' THEN 'Empty phone field'
        ELSE 'Phone available'
    END AS phone_status
FROM customers
WHERE email IS NOT NULL
ORDER BY customer_name;
```

### 4.2 COALESCE Function

**Example: Contact Information Hierarchy**
```sql
SELECT 
    customer_name,
    email,
    COALESCE(phone, 'No phone') AS contact_phone,
    COALESCE(city, 'Unknown city') AS customer_city,
    COALESCE(phone, email, 'No contact info') AS primary_contact
FROM customers
ORDER BY customer_name;
```

### 4.3 NULLIF Function

**Example: Avoiding Division by Zero**
```sql
SELECT 
    e.employee_name,
    e.department,
    e.salary,
    COALESCE(dept_stats.avg_salary, 0) AS dept_avg_salary,
    ROUND(
        e.salary / NULLIF(dept_stats.avg_salary, 0) * 100, 2
    ) AS salary_vs_dept_avg_percent
FROM employees e
LEFT JOIN (
    SELECT 
        department,
        AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department
) dept_stats ON e.department = dept_stats.department
ORDER BY e.department, e.salary DESC;
```

### 4.4 NULL in Aggregations

**Example: Customer Contact Analysis**
```sql
SELECT 
    COUNT(*) AS total_customers,
    COUNT(email) AS customers_with_email,
    COUNT(phone) AS customers_with_phone,
    COUNT(*) - COUNT(email) AS customers_without_email,
    COUNT(*) - COUNT(phone) AS customers_without_phone,
    ROUND(COUNT(email) * 100.0 / COUNT(*), 2) AS email_completion_rate,
    ROUND(COUNT(phone) * 100.0 / COUNT(*), 2) AS phone_completion_rate
FROM customers;
```

---

## 5. Subqueries & CTEs - Advanced Query Composition

Subqueries and CTEs enable complex data analysis and improve query readability.

### 5.1 Scalar Subqueries

**Example: Products Above Average Price**
```sql
SELECT 
    product_name,
    category,
    price,
    (SELECT AVG(price) FROM products) AS overall_avg_price,
    ROUND(price - (SELECT AVG(price) FROM products), 2) AS price_diff_from_avg
FROM products
WHERE price > (SELECT AVG(price) FROM products)
ORDER BY price_diff_from_avg DESC;
```

### 5.2 Table Subqueries

**Example: High-Value Customers**
```sql
SELECT 
    c.customer_name,
    c.email,
    customer_stats.total_orders,
    customer_stats.total_spent,
    customer_stats.avg_order_value
FROM customers c
INNER JOIN (
    SELECT 
        customer_id,
        COUNT(*) AS total_orders,
        SUM(total_amount) AS total_spent,
        ROUND(AVG(total_amount), 2) AS avg_order_value
    FROM orders
    WHERE status = 'COMPLETED'
    GROUP BY customer_id
    HAVING SUM(total_amount) > 1000
) customer_stats ON c.customer_id = customer_stats.customer_id
ORDER BY customer_stats.total_spent DESC;
```

### 5.3 Correlated Subqueries

**Example: Customers with Above-Average Order Values**
```sql
SELECT 
    c.customer_name,
    c.email,
    (SELECT COUNT(*) FROM orders o WHERE o.customer_id = c.customer_id) AS order_count,
    (SELECT ROUND(AVG(total_amount), 2) FROM orders o WHERE o.customer_id = c.customer_id) AS avg_order_value
FROM customers c
WHERE (
    SELECT AVG(total_amount)
    FROM orders o
    WHERE o.customer_id = c.customer_id
) > (
    SELECT AVG(total_amount) 
    FROM orders
)
ORDER BY avg_order_value DESC;
```

### 5.4 EXISTS vs IN

**Example: Active Customers (those who placed orders)**
```sql
-- Using EXISTS (generally more efficient)
SELECT 
    customer_name,
    email,
    signup_date
FROM customers c
WHERE EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.customer_id
      AND o.status = 'COMPLETED'
)
ORDER BY signup_date DESC;

-- Using IN (can be less efficient with large datasets)
SELECT 
    customer_name,
    email,
    signup_date
FROM customers
WHERE customer_id IN (
    SELECT DISTINCT customer_id
    FROM orders
    WHERE status = 'COMPLETED'
      AND customer_id IS NOT NULL
)
ORDER BY signup_date DESC;
```

### 5.5 Common Table Expressions (CTEs)

**Example: Sales Performance Analysis**
```sql
WITH monthly_sales AS (
    SELECT 
        YEAR(order_date) AS sales_year,
        MONTH(order_date) AS sales_month,
        SUM(total_amount) AS monthly_revenue,
        COUNT(*) AS monthly_orders
    FROM orders
    WHERE status = 'COMPLETED'
    GROUP BY YEAR(order_date), MONTH(order_date)
),
sales_with_growth AS (
    SELECT 
        sales_year,
        sales_month,
        monthly_revenue,
        monthly_orders,
        LAG(monthly_revenue, 1) OVER (ORDER BY sales_year, sales_month) AS prev_month_revenue,
        ROUND(
            (monthly_revenue - LAG(monthly_revenue, 1) OVER (ORDER BY sales_year, sales_month)) /
            NULLIF(LAG(monthly_revenue, 1) OVER (ORDER BY sales_year, sales_month), 0) * 100, 2
        ) AS growth_rate
    FROM monthly_sales
)
SELECT 
    sales_year,
    sales_month,
    monthly_revenue,
    monthly_orders,
    prev_month_revenue,
    growth_rate,
    CASE 
        WHEN growth_rate > 10 THEN 'High Growth'
        WHEN growth_rate > 0 THEN 'Positive Growth'
        WHEN growth_rate = 0 THEN 'No Growth'
        ELSE 'Decline'
    END AS growth_category
FROM sales_with_growth
ORDER BY sales_year, sales_month;
```

### 5.6 Recursive CTEs

**Example: Employee Hierarchy**
```sql
WITH RECURSIVE employee_hierarchy AS (
    -- Base case: Top-level managers (no manager)
    SELECT 
        employee_id,
        employee_name,
        manager_id,
        department,
        salary,
        0 AS level,
        CAST(employee_name AS CHAR(500)) AS hierarchy_path
    FROM employees
    WHERE manager_id IS NULL
    
    UNION ALL
    
    -- Recursive case: Employees with managers
    SELECT 
        e.employee_id,
        e.employee_name,
        e.manager_id,
        e.department,
        e.salary,
        eh.level + 1,
        CONCAT(eh.hierarchy_path, ' -> ', e.employee_name)
    FROM employees e
    INNER JOIN employee_hierarchy eh ON e.manager_id = eh.employee_id
    WHERE eh.level < 10  -- Prevent infinite recursion
)
SELECT 
    CONCAT(REPEAT('  ', level), employee_name) AS org_chart,
    department,
    salary,
    level,
    hierarchy_path
FROM employee_hierarchy
ORDER BY hierarchy_path;
```

---

## 6. Set Operations - Combining Result Sets

Set operations allow you to combine results from multiple queries.

### 6.1 UNION - Combining with Duplicate Removal

**Example: All Customer Communications**
```sql
SELECT 
    'Email' AS contact_type,
    email AS contact_info,
    customer_name
FROM customers
WHERE email IS NOT NULL

UNION

SELECT 
    'Phone' AS contact_type,
    phone AS contact_info,
    customer_name
FROM customers
WHERE phone IS NOT NULL
ORDER BY customer_name, contact_type;
```

### 6.2 UNION ALL - Combining with All Records

**Example: Transaction History from Multiple Sources**
```sql
SELECT 
    'ORDER' AS transaction_type,
    order_id AS transaction_id,
    customer_id,
    order_date AS transaction_date,
    total_amount
FROM orders
WHERE order_date >= '2024-01-01'

UNION ALL

SELECT 
    'REFUND' AS transaction_type,
    refund_id AS transaction_id,
    customer_id,
    refund_date AS transaction_date,
    -refund_amount AS total_amount
FROM refunds
WHERE refund_date >= '2024-01-01'
ORDER BY transaction_date DESC, customer_id;
```

### 6.3 INTERSECT - Common Records

**Example: Customers Who Bought Both Electronics and Books**
```sql
-- Customers who bought Electronics
SELECT DISTINCT c.customer_id, c.customer_name
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id
INNER JOIN order_details od ON o.order_id = od.order_id
INNER JOIN products p ON od.product_id = p.product_id
WHERE p.category = 'Electronics'

INTERSECT

-- Customers who bought Books
SELECT DISTINCT c.customer_id, c.customer_name
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id
INNER JOIN order_details od ON o.order_id = od.order_id
INNER JOIN products p ON od.product_id = p.product_id
WHERE p.category = 'Books';
```

### 6.4 EXCEPT/MINUS - Difference Between Sets

**Example: Customers Who Bought Electronics but Not Books**
```sql
-- Customers who bought Electronics
SELECT DISTINCT c.customer_id, c.customer_name
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id
INNER JOIN order_details od ON o.order_id = od.order_id
INNER JOIN products p ON od.product_id = p.product_id
WHERE p.category = 'Electronics'

EXCEPT

-- Customers who bought Books
SELECT DISTINCT c.customer_id, c.customer_name
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id
INNER JOIN order_details od ON o.order_id = od.order_id
INNER JOIN products p ON od.product_id = p.product_id
WHERE p.category = 'Books';
```

---

## 7. Advanced Filtering - WHERE vs HAVING

Understanding when to use WHERE vs HAVING is crucial for efficient querying.

### 7.1 WHERE Clause (Row-Level Filtering)

Filters individual rows **before** grouping occurs.

**Example: Department Analysis for Recent Hires**
```sql
SELECT 
    department,
    COUNT(*) AS employee_count,
    ROUND(AVG(salary), 2) AS avg_salary,
    MIN(hire_date) AS earliest_hire,
    MAX(hire_date) AS latest_hire
FROM employees
WHERE hire_date >= '2023-01-01'  -- Filter rows before grouping
  AND status = 'ACTIVE'
GROUP BY department
ORDER BY avg_salary DESC;
```

### 7.2 HAVING Clause (Group-Level Filtering)

Filters groups **after** grouping and aggregation.

**Example: High-Performing Departments**
```sql
SELECT 
    department,
    COUNT(*) AS employee_count,
    ROUND(AVG(salary), 2) AS avg_salary,
    SUM(salary) AS total_payroll
FROM employees
WHERE status = 'ACTIVE'
GROUP BY department
HAVING COUNT(*) >= 5           -- Filter groups after aggregation
   AND AVG(salary) > 75000
ORDER BY avg_salary DESC;
```

### 7.3 WHERE and HAVING Together

**Example: Customer Loyalty Analysis**
```sql
SELECT 
    c.customer_name,
    c.signup_date,
    COUNT(o.order_id) AS total_orders,
    SUM(o.total_amount) AS total_spent,
    ROUND(AVG(o.total_amount), 2) AS avg_order_value
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id
WHERE o.status = 'COMPLETED'                    -- Filter rows first
  AND o.order_date >= '2024-01-01'
  AND c.status = 'ACTIVE'
GROUP BY c.customer_id, c.customer_name, c.signup_date
HAVING COUNT(o.order_id) >= 3                  -- Filter groups after
   AND SUM(o.total_amount) > 500
ORDER BY total_spent DESC;
```

### 7.4 Performance Comparison

```sql
-- EFFICIENT: Filter early with WHERE
SELECT 
    category,
    COUNT(*) AS product_count,
    ROUND(AVG(price), 2) AS avg_price
FROM products
WHERE price > 0              -- Reduces rows before grouping
  AND stock_quantity > 0
GROUP BY category
HAVING COUNT(*) > 10;

-- LESS EFFICIENT: Could filter earlier
SELECT 
    category,
    COUNT(*) AS product_count,
    ROUND(AVG(price), 2) AS avg_price
FROM products
GROUP BY category
HAVING COUNT(*) > 10
   AND MIN(price) > 0        -- This filtering could be done earlier
   AND MIN(stock_quantity) > 0;
```

---

## 8. String Functions - Text Processing

String manipulation is essential for data cleaning and formatting.

### 8.1 Basic String Functions

| Function | Description | Example | Result |
|----------|-------------|---------|--------|
| `CONCAT()` | Join strings | `CONCAT('Hello', ' ', 'World')` | `'Hello World'` |
| `CONCAT_WS()` | Join with separator | `CONCAT_WS(', ', 'John', 'Doe')` | `'John, Doe'` |
| `LENGTH()` | String length | `LENGTH('Hello')` | `5` |
| `UPPER()` / `LOWER()` | Change case | `UPPER('Hello')` | `'HELLO'` |
| `TRIM()` | Remove whitespace | `TRIM('  Hello  ')` | `'Hello'` |

**Example: Customer Data Cleaning**
```sql
SELECT 
    customer_id,
    TRIM(UPPER(customer_name)) AS clean_name,
    LOWER(TRIM(email)) AS clean_email,
    CONCAT(
        UPPER(LEFT(customer_name, 1)),
        LOWER(SUBSTRING(customer_name, 2))
    ) AS formatted_name,
    LENGTH(customer_name) AS name_length
FROM customers
WHERE customer_name IS NOT NULL
ORDER BY clean_name;
```

### 8.2 String Extraction Functions

| Function | Description | Example | Result |
|----------|-------------|---------|--------|
| `SUBSTRING()` | Extract substring | `SUBSTRING('database', 5, 4)` | `'base'` |
| `LEFT()` / `RIGHT()` | Extract from ends | `LEFT('database', 4)` | `'data'` |
| `LOCATE()` | Find position | `LOCATE('base', 'database')` | `5` |
| `REPLACE()` | Replace substring | `REPLACE('a-b-c', '-', ' ')` | `'a b c'` |

**Example: Email Domain Analysis**
```sql
SELECT 
    SUBSTRING(email, LOCATE('@', email) + 1) AS email_domain,
    COUNT(*) AS customer_count,
    ROUND(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM customers WHERE email IS NOT NULL), 2) AS percentage
FROM customers
WHERE email IS NOT NULL
  AND LOCATE('@', email) > 0
GROUP BY SUBSTRING(email, LOCATE('@', email) + 1)
ORDER BY customer_count DESC;
```

### 8.3 Pattern Matching and Validation

**Example: Phone Number Validation and Formatting**
```sql
SELECT 
    customer_name,
    phone AS original_phone,
    CASE 
        WHEN phone REGEXP '^[0-9]{10}$' THEN 
            CONCAT('(', LEFT(phone, 3), ') ', SUBSTRING(phone, 4, 3), '-', RIGHT(phone, 4))
        WHEN phone REGEXP '^[0-9]{3}-[0-9]{3}-[0-9]{4}$' THEN phone
        WHEN phone REGEXP '^\\([0-9]{3}\\) [0-9]{3}-[0-9]{4}$' THEN phone
        ELSE 'Invalid Format'
    END AS formatted_phone,
    CASE 
        WHEN phone REGEXP '^[0-9]{10}$' OR 
             phone REGEXP '^[0-9]{3}-[0-9]{3}-[0-9]{4}$' OR 
             phone REGEXP '^\\([0-9]{3}\\) [0-9]{3}-[0-9]{4}$' THEN 'Valid'
        ELSE 'Invalid'
    END AS validation_status
FROM customers
WHERE phone IS NOT NULL
ORDER BY validation_status, customer_name;
```

### 8.4 String Aggregation

**Example: Product Catalog Summary**
```sql
SELECT 
    GROUP_CONCAT(DISTINCT category ORDER BY category SEPARATOR ', ') AS categories,
    GROUP_CONCAT(DISTINCT brand ORDER BY brand SEPARATOR ', ') AS brands,
    GROUP_CONCAT(DISTINCT color ORDER BY color SEPARATOR ', ') AS colors,
    GROUP_CONCAT(DISTINCT size ORDER BY size SEPARATOR ', ') AS sizes,
    GROUP_CONCAT(DISTINCT material ORDER BY material SEPARATOR ', ') AS materials,
    GROUP_CONCAT(DISTINCT material ORDER BY material SEPARATOR ', ') AS materials,
    GROUP_CONCAT(DISTINCT material ORDER BY material SEPARATOR ', ') AS materials,
```