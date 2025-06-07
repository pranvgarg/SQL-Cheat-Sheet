# SQL Industrial Questions and Solutions

This document provides a comprehensive collection of real-world SQL questions encountered in technical interviews and industry scenarios. Each question is presented with a clear business context, table structures, expected output, and a detailed solution.

## Table of Contents

1. [E-commerce Analytics](#e-commerce-analytics)
2. [Financial Reporting](#financial-reporting)
3. [Supply Chain Management](#supply-chain-management)
4. [Healthcare Analytics](#healthcare-analytics)
5. [Customer Segmentation](#customer-segmentation)
6. [SQL Interview Questions & Solutions](#sql-interview-questions-solutions)
7. [Advanced Analytics](#advanced-analytics)

## How to Use This Guide

- Each section contains related SQL problems
- Problems are organized by difficulty and domain
- Solutions include explanations of key concepts used
- Table structures are provided for context

## E-commerce Analytics

### Question 1: Customer Purchase Funnel Analysis

**Interviewer's Context:**
"Let's analyze our e-commerce platform's purchase funnel. We track user interactions through different stages: viewing a product, adding it to cart, starting checkout, and completing the purchase. We need to understand where we're losing potential customers in this journey."

**Data Available:**

- We have a `user_events` table that logs all user interactions
- Each event has a timestamp, session ID, and event type
- The funnel steps are tracked as distinct event types
- We're interested in the last 7 days of data

**Table Structure:**

```sql
CREATE TABLE user_events (
    event_id INT,
    session_id VARCHAR(50),      -- Unique identifier for user session
    user_id INT,                -- User identifier (can be NULL for non-logged-in users)
    event_type VARCHAR(50),     -- 'product_view', 'add_to_cart', 'checkout_started', 'purchase_completed'
    event_date TIMESTAMP,       -- When the event occurred
    product_id INT,             -- The product involved (if applicable)
    -- other relevant columns
);
```

**Sample Data:**

```sql
-- Sample data for user_events table
INSERT INTO user_events VALUES
    (1, 'sess_abc', 123, 'product_view', '2023-05-01 10:15:22', 101),
    (2, 'sess_abc', 123, 'add_to_cart', '2023-05-01 10:16:05', 101),
    (3, 'sess_abc', 123, 'checkout_started', '2023-05-01 10:20:30', 101),
    (4, 'sess_abc', 123, 'purchase_completed', '2023-05-01 10:22:45', 101),
    (5, 'sess_def', 456, 'product_view', '2023-05-02 14:30:00', 205),
    (6, 'sess_def', 456, 'add_to_cart', '2023-05-02 14:31:15', 205),
    (7, 'sess_ghi', NULL, 'product_view', '2023-05-03 09:05:33', 101);
```

**Question for the Candidate:**

"Can you help us calculate the conversion rates between each step of our purchase funnel for the last 7 days? We want to see what percentage of users move from one step to the next in the funnel. The funnel steps in order are: product view → add to cart → checkout start → purchase completion. We should get one row showing the total sessions and the conversion rates between each step."

**Expected Output Format:**

```sql
total_sessions | view_rate | cart_add_rate | checkout_rate | conversion_rate
--------------|-----------|---------------|---------------|----------------
     10,245  |   85.42   |     42.15     |     68.30     |     75.25
```

**Clarifications (if asked by candidate):**

- A 'session' is a unique visit to the site
- Users can view multiple products in a session, but we count unique sessions per step
- We're only interested in the first occurrence of each event type per session
- The rates should be calculated as percentages (0-100)
- Round all percentages to 2 decimal places

**Table Structure:**

```sql
CREATE TABLE user_events (
    event_id INT,
    session_id VARCHAR(50),
    user_id INT,
    event_type VARCHAR(50),
    event_date TIMESTAMP,
    -- other relevant columns
)
```

**Expected Output:**

```sql
total_sessions | view_rate | cart_add_rate | checkout_rate | conversion_rate
--------------|-----------|---------------|---------------|----------------
     10,245  |   85.42   |     42.15     |     68.30     |     75.25
```

**Solution:**
```sql
WITH funnel AS (
    SELECT
        COUNT(DISTINCT session_id) as total_sessions,
        COUNT(DISTINCT CASE WHEN event_type = 'product_view' THEN session_id END) as viewed_product,
        COUNT(DISTINCT CASE WHEN event_type = 'add_to_cart' THEN session_id END) as added_to_cart,
        COUNT(DISTINCT CASE WHEN event_type = 'checkout_started' THEN session_id END) as started_checkout,
        COUNT(DISTINCT CASE WHEN event_type = 'purchase_completed' THEN session_id END) as completed_purchase
    FROM user_events
    WHERE event_date BETWEEN CURRENT_DATE - INTERVAL '7 days' AND CURRENT_DATE
)
SELECT
    total_sessions,
    ROUND(viewed_product * 100.0 / NULLIF(total_sessions, 0), 2) as view_rate,
    ROUND(added_to_cart * 100.0 / NULLIF(viewed_product, 0), 2) as cart_add_rate,
    ROUND(started_checkout * 100.0 / NULLIF(added_to_cart, 0), 2) as checkout_rate,
    ROUND(completed_purchase * 100.0 / NULLIF(started_checkout, 0), 2) as conversion_rate
FROM funnel;
```

**Key Concepts:**
- **CTE (Common Table Expression)**: Creates a temporary result set for the funnel metrics
- **CASE WHEN**: Conditional counting of events
- **NULLIF**: Prevents division by zero errors
- **ROUND**: Formats percentages to 2 decimal places
- **Date Filtering**: Focuses on the last 7 days of data

**Business Insight:**  
This query helps identify the weakest points in the conversion funnel. For example, if the cart add rate is low compared to product views, the product pages might need optimization.

## Financial Reporting

### Question 2: Monthly Revenue Growth Analysis

**Interviewer's Context:**
"We need to analyze our revenue trends to understand our business performance better. Specifically, we want to track how our monthly revenue is growing or shrinking compared to the previous month. This will help us identify seasonal patterns and measure the impact of our business initiatives."

**Data Available:**
- We have an `orders` table containing all customer orders
- Each order has an order date and total amount
- We need to analyze data for the current calendar year
- Some months might have zero revenue (no orders)

**Table Structure:**

```sql
orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    order_date TIMESTAMP,      -- When the order was placed
    total_amount DECIMAL(10,2), -- Total order amount in USD
    status VARCHAR(20),        -- 'completed', 'returned', 'cancelled', etc.
    -- other order details
)
```

**Sample Data:**

```sql
| order_id | customer_id |       order_date       | total_amount |   status   |
|----------|-------------|------------------------|--------------|------------|
|    1001 |        4567 | 2023-01-15 10:30:00   |      129.99  | 'completed'|
|    1002 |        2345 | 2023-01-18 14:22:10   |       89.50  | 'completed'|
|    1003 |        4567 | 2023-02-02 09:15:33   |      210.00  | 'completed'|
|    1004 |        1234 | 2023-02-14 16:45:22   |       45.99  | 'returned' |
|    1005 |        7890 | 2023-03-01 11:20:15   |      175.50  | 'completed'|
|    1006 |        3456 | 2023-03-15 13:05:44   |      299.99  | 'completed'|
|    1007 |        2345 | 2023-03-20 17:30:00   |       59.99  | 'completed'|
```

**Question for the Candidate:**

"Can you help us analyze our monthly revenue growth for the current year? We'd like to see:

1. The total revenue for each month
2. The month-over-month growth percentage
3. The absolute change in revenue compared to the previous month

Please exclude any orders that were returned or cancelled. We should have one row per month, ordered chronologically, with the most recent month last. For the first month, the growth percentage and absolute change should be NULL since there's no previous month to compare with."

**Expected Output Format:**

```sql
  year  | month | monthly_revenue | revenue_growth_pct | revenue_change_abs
--------|-------|-----------------|--------------------|------------------
  2023  |   1   |     219.49      |        NULL        |       NULL
  2023  |   2   |     210.00      |       -4.32        |       -9.49
  2023  |   3   |     535.48      |      155.01        |      325.48
```

**Clarifications (if asked by candidate):**
- Round monetary values to 2 decimal places
- Round percentage changes to 2 decimal places
- Only include orders with status 'completed'
- If a month has no orders, show it with $0 revenue
- The growth percentage formula is: ((current_month - previous_month) / previous_month) * 100

**Solution Approach:**

1. First, calculate the monthly revenue by filtering for completed orders and grouping by year and month
2. Use window functions to access the previous month's revenue for comparison
3. Calculate both the absolute change and percentage growth
4. Handle edge cases like the first month (no previous month to compare)

**SQL Solution:**

```sql
WITH monthly_revenue AS (
    SELECT 
        EXTRACT(YEAR FROM order_date) AS year,
        EXTRACT(MONTH FROM order_date) AS month,
        ROUND(SUM(total_amount)::numeric, 2) AS monthly_revenue
    FROM 
        orders
    WHERE 
        status = 'completed'
        AND EXTRACT(YEAR FROM order_date) = EXTRACT(YEAR FROM CURRENT_DATE)
    GROUP BY 
        EXTRACT(YEAR FROM order_date),
        EXTRACT(MONTH FROM order_date)
)
SELECT 
    year,
    month,
    monthly_revenue,
    ROUND(
        ((monthly_revenue - LAG(monthly_revenue) OVER (ORDER BY year, month)) / 
        LAG(monthly_revenue) OVER (ORDER BY year, month) * 100)::numeric, 
        2
    ) AS revenue_growth_pct,
    ROUND(
        (monthly_revenue - LAG(monthly_revenue) OVER (ORDER BY year, month))::numeric, 
        2
    ) AS revenue_change_abs
FROM 
    monthly_revenue
ORDER BY 
    year, month;
```

**Key Learning Points:**
- Using Common Table Expressions (CTEs) for better query organization
- Window functions (LAG) to access data from previous rows
- Handling NULL values for the first month's comparison
- Proper type casting for accurate decimal calculations
- Filtering for only completed orders to get accurate revenue
- Grouping by both year and month to handle multi-year data correctly

**Potential Follow-up Questions:**
1. How would you modify this query to show year-over-year growth instead of month-over-month?
2. How would you handle months with no orders in the results?
3. How could you optimize this query for a very large orders table?

**Alternative Solutions:**
- Using self-joins instead of window functions (less efficient)
- Creating a date dimension table for handling months with no orders
- Using a stored procedure for more complex business logic

### Expected Output

```sql
  year  | month | monthly_revenue | revenue_growth_pct | revenue_change_abs
--------|-------|-----------------|--------------------|------------------
  2023  |   1   |    125430.00    |        NULL        |       NULL
  2023  |   2   |    138750.00    |       10.63        |      13320.00
  2023  |   3   |    145200.00    |        4.65        |       6450.00
  2023  |   4   |          0.00   |     -100.00        |    -145200.00
  2023  |   5   |     98500.00    |         -          |      98500.00
  2023  |   6   |    112300.00    |       14.01        |      13800.00
  2023  |   7   |    125000.00    |       11.31        |      12700.00
  2023  |   8   |    140000.00    |       12.00        |      15000.00
  2023  |   9   |    135000.00    |       -3.57        |      -5000.00
  2023  |  10   |    150000.00    |       11.11        |      15000.00
  2023  |  11   |    175000.00    |       16.67        |      25000.00
  2023  |  12   |    250000.00    |       42.86        |      75000.00
```

### Notes on the Output

- The first month (January 2023) shows NULL for growth metrics as there's no previous month to compare
- April 2023 shows $0 revenue (no orders that month)
- The growth percentage is calculated based on the previous month's revenue
- All monetary values are rounded to 2 decimal places
- The data shows a seasonal pattern with higher revenue at year-end

**Solution:**
```sql
WITH monthly_revenue AS (
    SELECT
        DATE_TRUNC('month', order_date) as month,
        SUM(amount) as revenue
    FROM orders
    WHERE order_date >= DATE_TRUNC('year', CURRENT_DATE)
      AND status = 'completed'
    GROUP BY 1
)
SELECT
    TO_CHAR(month, 'YYYY-MM') as month,
    revenue,
    LAG(revenue) OVER (ORDER BY month) as prev_month_revenue,
    ROUND(
        (revenue - LAG(revenue) OVER (ORDER BY month)) * 100.0 / 
        NULLIF(LAG(revenue) OVER (ORDER BY month), 0), 
        2
    ) as mom_growth_pct
FROM monthly_revenue
ORDER BY month;
```

**Key Concepts:**
- **DATE_TRUNC**: Truncates timestamps to the specified precision (month in this case)
- **Window Functions**: LAG() accesses data from previous rows
- **NULLIF**: Prevents division by zero errors
- **TO_CHAR**: Formats dates for better readability

**Business Insight:**  
This query helps identify seasonal patterns and growth trends. For example, if March shows a significant growth percentage, it might indicate the success of a marketing campaign or seasonal demand.

## Financial Reporting - Advanced Metrics

### Question 3: Monthly Revenue Growth Analysis

**Business Problem:** Calculate month-over-month revenue growth percentage.

```sql
WITH monthly_revenue AS (
    SELECT
        DATE_TRUNC('month', order_date) as month,
        SUM(amount) as revenue
    FROM orders
    WHERE order_date >= DATE_TRUNC('year', CURRENT_DATE)
    GROUP BY 1
)
SELECT
    TO_CHAR(month, 'YYYY-MM') as month,
    revenue,
    LAG(revenue) OVER (ORDER BY month) as prev_month_revenue,
    ROUND(
        (revenue - LAG(revenue) OVER (ORDER BY month)) * 100.0 / 
        LAG(revenue) OVER (ORDER BY month), 
        2
    ) as mom_growth_pct
FROM monthly_revenue
ORDER BY month;
```

### Question 4: Customer Lifetime Value (CLV)

**Business Context:**  
Understanding customer value is crucial for marketing budget allocation and customer acquisition strategy. CLV helps identify the most valuable customer segments.

**Problem Statement:**  
Calculate the 90-day Customer Lifetime Value (CLV) by acquisition channel, including key metrics like total customers, orders, revenue, and average order value.

**Table Structures:**

```sql
users (
    id INT,
    acquisition_channel VARCHAR(50),
    signup_date DATE,
    -- other user fields
)

orders (
    id INT,
    customer_id INT,
    order_date TIMESTAMP,
    amount DECIMAL(10,2),
    status VARCHAR(20)
    -- other order fields
)
```

**Expected Output:**

```sql
-- Expected output format for Customer Lifetime Value query
acquisition_channel | total_customers | total_orders | total_revenue | avg_orders | avg_revenue | estimated_90day_clv
-------------------|-----------------|--------------|---------------|------------|-------------|-------------------
Organic Search    | 1,250          | 3,450        | 172,500.00    | 2.76       | 138.00      | 34.50
Paid Social      | 980            | 2,450        | 134,750.00    | 2.50       | 137.50      | 34.38
Email            | 1,100          | 2,750        | 137,500.00    | 2.50       | 125.00      | 31.25
```

**Solution:**

```sql
WITH customer_metrics AS (
    SELECT
        u.acquisition_channel,
        COUNT(DISTINCT o.customer_id) as total_customers,
        COUNT(DISTINCT o.order_id) as total_orders,
        SUM(o.amount) as total_revenue,
        COUNT(DISTINCT o.order_id) * 1.0 / NULLIF(COUNT(DISTINCT o.customer_id), 0) as avg_orders_per_customer,
        SUM(o.amount) / NULLIF(COUNT(DISTINCT o.customer_id), 0) as avg_revenue_per_customer
    FROM users u
    JOIN orders o ON u.id = o.customer_id
    WHERE o.order_date BETWEEN CURRENT_DATE - 90 AND CURRENT_DATE
      AND o.status = 'completed'
    GROUP BY 1
)

SELECT
    acquisition_channel,
    total_customers,
    total_orders,
    ROUND(total_revenue, 2) as total_revenue,
    ROUND(avg_orders_per_customer, 2) as avg_orders_per_customer,
    ROUND(avg_revenue_per_customer, 2) as avg_revenue_per_customer,
    ROUND(avg_revenue_per_customer * 0.25, 2) as estimated_90day_clv -- 25% profit margin
FROM customer_metrics
ORDER BY avg_revenue_per_customer DESC;
```

**Key Concepts:**
- **NULLIF**: Prevents division by zero errors
- **Aggregation**: Calculates key metrics by acquisition channel
- **Business Logic**: Applies 25% profit margin to estimate CLV
- **Date Filtering**: Focuses on the last 90 days of data

**Business Insight:**  
This analysis helps identify which acquisition channels bring in the most valuable customers. For example, if Organic Search has a higher CLV, it might be worth investing more in SEO and content marketing.

## Supply Chain Management

### Question 5: Inventory Turnover Ratio

**Business Context:**  
Inventory turnover ratio measures how often a company sells and replaces its inventory during a period. This metric is crucial for supply chain optimization and working capital management.

**Problem Statement:**  
Calculate the monthly inventory turnover ratio by product category for the current year. The ratio is calculated as: (Units Sold) / (Average Inventory).

**Table Structures:**

```sql
inventory (
    id INT,
    product_id INT,
    date DATE,
    quantity_on_hand INT,
    warehouse_id INT
)

-- Table structure for product catalog
products (
    id INT,
    name VARCHAR(100),
    category VARCHAR(50),
    price DECIMAL(10,2)
)

-- Table structure for order line items
order_items (
    id INT,
    order_id INT,
    product_id INT,
    quantity INT,
    created_at TIMESTAMP
)
```

**Expected Output:**

```sql
-- Expected output format for Inventory Turnover query
  category   |  month  | inventory_turnover_ratio
-------------|---------|-------------------------
 Electronics | 2023-01 |          4.25
 Furniture   | 2023-01 |          1.80
 Electronics | 2023-02 |          5.10
 Furniture   | 2023-02 |          2.15
```

**Solution:**

```sql
WITH monthly_inventory AS (
    SELECT
        p.category,
        DATE_TRUNC('month', i.date) as month,
        AVG(i.quantity_on_hand) as avg_inventory,
        COALESCE(SUM(oi.quantity), 0) as units_sold
    FROM inventory i
    JOIN products p ON i.product_id = p.id
    LEFT JOIN order_items oi 
        ON p.id = oi.product_id 
        AND oi.created_at >= DATE_TRUNC('month', i.date)
        AND oi.created_at < DATE_TRUNC('month', i.date) + INTERVAL '1 month'
    WHERE i.date >= DATE_TRUNC('year', CURRENT_DATE)
    GROUP BY 1, 2
)
SELECT
    category,
    TO_CHAR(month, 'YYYY-MM') as month,
    ROUND(
        units_sold / NULLIF(avg_inventory, 0), 
        2
    ) as inventory_turnover_ratio
FROM monthly_inventory
WHERE avg_inventory > 0  -- Exclude categories with no inventory
ORDER BY category, month;
```

**Key Concepts:**
- **DATE_TRUNC**: Groups data by month
- **LEFT JOIN**: Ensures all inventory is included, even if no sales occurred
- **NULLIF**: Prevents division by zero
- **COALESCE**: Handles months with no sales
- **Date Range Filtering**: Focuses on the current year

**Business Insight:**  
A higher ratio indicates faster inventory turnover, which is generally positive. However, extremely high ratios might indicate stockouts, while low ratios may suggest overstocking or obsolescence. For example, if Electronics has a ratio of 5.10, it means the inventory was sold and replaced 5.1 times in February 2023.

### Question 6: Supplier Performance Analysis
**Business Problem:** Identify suppliers with late shipments.


```sql
-- Supplier performance analysis query
SELECT
    s.supplier_name,
    COUNT(DISTINCT po.id) as total_orders,
    SUM(CASE 
            WHEN po.actual_delivery_date > po.expected_delivery_date 
            THEN 1 ELSE 0 
        END) as late_orders,
    ROUND(AVG(
        CASE 
            WHEN po.actual_delivery_date > po.expected_delivery_date 
            THEN DATE_PART('day', po.actual_delivery_date - po.expected_delivery_date)
            ELSE 0 
        END
    ), 2) as avg_days_late,
    ROUND(SUM(
        CASE 
            WHEN po.actual_delivery_date > po.expected_delivery_date 
            THEN 1 ELSE 0 
        END) * 100.0 / COUNT(DISTINCT po.id), 2) as late_order_percentage
FROM purchase_orders po
JOIN suppliers s ON po.supplier_id = s.id
WHERE po.order_date >= CURRENT_DATE - INTERVAL '6 months'
GROUP BY s.supplier_name
HAVING COUNT(DISTINCT po.id) >= 5
ORDER BY late_order_percentage DESC
LIMIT 10;
```

## Healthcare Analytics

### Question 7: Patient Readmission Rate
**Business Problem:** Calculate 30-day readmission rates by department.


```sql
-- Patient readmission analysis query
WITH readmissions AS (
    SELECT
        a1.patient_id,
        a1.discharge_date,
        a2.admission_date as readmission_date,
        a1.department,
        DATEDIFF(day, a1.discharge_date, a2.admission_date) as days_to_readmit
    FROM admissions a1
    JOIN admissions a2 
        ON a1.patient_id = a2.patient_id
        AND a1.discharge_date < a2.admission_date
        AND DATEDIFF(day, a1.discharge_date, a2.admission_date) <= 30
)

-- Calculate readmission rates by department
SELECT
    department,
    COUNT(DISTINCT patient_id) as total_patients,
    COUNT(DISTINCT CASE WHEN days_to_readmit <= 30 THEN patient_id END) as readmitted_patients,
    ROUND(COUNT(DISTINCT CASE WHEN days_to_readmit <= 30 THEN patient_id END) * 100.0 / 
          COUNT(DISTINCT patient_id), 2) as readmission_rate
FROM readmissions
GROUP BY department
ORDER BY readmission_rate DESC;
```

## Customer Segmentation

### Question 8: RFM Analysis
**Business Problem:** Segment customers using RFM (Recency, Frequency, Monetary) analysis.

```sql
WITH customer_rfm AS (
    SELECT
        customer_id,
        MAX(order_date) as last_order_date,
        COUNT(DISTINCT order_id) as frequency,
        SUM(amount) as monetary_value,
        DATEDIFF(day, MAX(order_date), CURRENT_DATE) as recency_days
    FROM orders
    WHERE order_date >= CURRENT_DATE - 365
    GROUP BY customer_id
    HAVING COUNT(DISTINCT order_id) >= 2
),
rfm_scores AS (
    SELECT
        customer_id,
        recency_days,
        frequency,
        monetary_value,
        NTILE(5) OVER (ORDER BY recency_days DESC) as r_score,
        NTILE(5) OVER (ORDER BY frequency) as f_score,
        NTILE(5) OVER (ORDER BY monetary_value) as m_score
    FROM customer_rfm
)
SELECT
    customer_id,
    recency_days,
    frequency,
    monetary_value,
    r_score,
    f_score,
    m_score,
    CONCAT(r_score, f_score, m_score) as rfm_cell,
    CASE
        WHEN r_score >= 4 AND f_score >= 4 AND m_score >= 4 THEN 'Champions'
        WHEN r_score >= 3 AND f_score >= 3 AND m_score >= 3 THEN 'Loyal Customers'
        WHEN r_score >= 3 AND f_score >= 1 AND m_score >= 2 THEN 'Potential Loyalists'
        WHEN r_score >= 4 AND f_score <= 2 AND m_score <= 2 THEN 'New Customers'
        WHEN r_score >= 2 AND f_score <= 2 AND m_score <= 2 THEN 'At Risk'
        WHEN r_score <= 2 AND f_score >= 3 AND m_score >= 3 THEN 'Cannot Lose Them'
        WHEN r_score <= 2 AND f_score <= 2 AND m_score >= 3 THEN 'Hibernating'
        ELSE 'About to Sleep'
    END as rfm_segment
FROM rfm_scores
ORDER BY r_score + f_score + m_score DESC;
```

### Question 9: Cohort Retention Analysis
**Business Problem:** Analyze customer retention by acquisition cohort.

```sql
WITH first_purchases AS (
    SELECT
        customer_id,
        DATE_TRUNC('month', MIN(order_date)) as cohort_month
    FROM orders
    GROUP BY 1
),
cohort_size AS (
    SELECT
        cohort_month,
        COUNT(DISTINCT customer_id) as num_customers
    FROM first_purchases
    GROUP BY 1
),
retention_data AS (
    SELECT
        fp.cohort_month,
        DATE_TRUNC('month', o.order_date) as order_month,
        EXTRACT(MONTH FROM AGE(DATE_TRUNC('month', o.order_date), fp.cohort_month)) as month_number,
        COUNT(DISTINCT o.customer_id) as retained_customers
    FROM orders o
    JOIN first_purchases fp ON o.customer_id = fp.customer_id
    GROUP BY 1, 2, 3
)
SELECT
    TO_CHAR(r.cohort_month, 'YYYY-MM') as cohort,
    r.month_number,
    c.num_customers as cohort_size,
    r.retained_customers,
    ROUND(r.retained_customers * 100.0 / c.num_customers, 2) as retention_rate
FROM retention_data r
JOIN cohort_size c ON r.cohort_month = c.cohort_month
WHERE r.cohort_month >= DATE_TRUNC('year', CURRENT_DATE) - INTERVAL '1 year'
ORDER BY r.cohort_month, r.month_number;
```

### Question 10: Churn Prediction
**Business Problem:** Identify customers at risk of churning.

```sql
WITH user_activity AS (
    SELECT
        user_id,
        MAX(activity_date) as last_activity_date,
        COUNT(DISTINCT CASE WHEN activity_date >= CURRENT_DATE - 30 THEN session_id END) as sessions_30d,
        COUNT(DISTINCT CASE WHEN activity_date >= CURRENT_DATE - 60 AND activity_date < CURRENT_DATE - 30 THEN session_id END) as sessions_60d_30d,
        MAX(activity_date) - MIN(activity_date) as days_active,
        DATEDIFF(day, MAX(activity_date), CURRENT_DATE) as days_since_last_activity
    FROM user_sessions
    WHERE activity_date >= CURRENT_DATE - 90
    GROUP BY 1
),
payment_status AS (
    SELECT
        user_id,
        MAX(due_date) as last_due_date,
        MAX(CASE WHEN status = 'overdue' THEN 1 ELSE 0 END) as has_overdue_payments
    FROM subscriptions
    WHERE status IN ('active', 'overdue')
    GROUP BY 1
),
churn_risk AS (
    SELECT
        u.user_id,
        u.last_activity_date,
        u.days_since_last_activity,
        u.sessions_30d,
        u.sessions_60d_30d,
        u.days_active,
        ps.has_overdue_payments,
        DATEDIFF(day, ps.last_due_date, CURRENT_DATE) as days_since_last_due,
        -- Churn risk score (higher = more likely to churn)
        CASE
            WHEN u.days_since_last_activity > 30 THEN 1.0
            WHEN u.days_since_last_activity > 14 THEN 0.8
            WHEN u.days_since_last_activity > 7 THEN 0.6
            WHEN u.sessions_30d < u.sessions_60d_30d * 0.5 THEN 0.7
            WHEN ps.has_overdue_payments = 1 AND ps.days_since_last_due > 14 THEN 0.9
            WHEN u.days_active < 30 THEN 0.3
            ELSE 0.2
        END as churn_risk_score
    FROM user_activity u
    LEFT JOIN payment_status ps ON u.user_id = ps.user_id
    WHERE u.days_since_last_activity <= 60  -- Only consider recently active users
)
SELECT
    cr.user_id,
    u.email,
    u.signup_date,
    cr.churn_risk_score,
    CASE
        WHEN cr.churn_risk_score >= 0.8 THEN 'High Risk'
        WHEN cr.churn_risk_score >= 0.5 THEN 'Medium Risk'
        ELSE 'Low Risk'
    END as churn_risk_category,
    cr.last_activity_date,
    cr.days_since_last_activity,
    cr.sessions_30d,
    cr.has_overdue_payments,
    cr.days_since_last_due
FROM churn_risk cr
JOIN users u ON cr.user_id = u.id
WHERE cr.churn_risk_score >= 0.5  -- Only show medium and high risk users
ORDER BY cr.churn_risk_score DESC
LIMIT 100;
```

These examples demonstrate how SQL can be used to solve complex business problems across various industries. The queries include advanced SQL features like window functions, common table expressions (CTEs), complex joins, and conditional logic that are commonly used in industrial settings.


### SQL Interview Questions & Solutions

### Question 1: Employee Filtering by Salary and Tenure

**Interview Context:**
"We have an Employee table with salary and tenure information. As a data analyst, you need to identify high-performing employees who are still relatively new to the company. Specifically, I want you to find employees who earn more than $2000 and have been with us for less than 10 months. This will help us understand our high-potential early-career talent."

**Table Structure:**
```sql
Employee Table:
employee_id | name    | salary | months
1          | Alice   | 2500   | 8
2          | Bob     | 1500   | 12
3          | Charlie | 2800   | 9
```

**Expected Output:**
```
name
Alice
Charlie
```

**Solution:**
```sql
SELECT
    name                                   -- Select only the employee's name
FROM
    Employee
WHERE
    salary > 2000                          -- Filter for salary strictly above 2000
    AND months < 10                        -- Filter for tenure less than 10 months
ORDER BY
    employee_id ASC;                       -- Order by employee_id ascending
```

---

## Question 2: Finding Shortest and Longest City Names

**Interview Context:**
"You're working with geographical data for our logistics team. They need to optimize their systems and want to understand the range of city name lengths they're dealing with. Find the city with the shortest name and the city with the longest name. If there are ties, resolve them alphabetically."

**Table Structure:**
```sql
STATION Table:
CITY
Newark
NY
Sacramento
Chicago
```

**Expected Output:**
```
CITY       | name_length
NY         | 2
Sacramento | 10
```

**Solution:**
```sql
(
  SELECT
    CITY,
    CHAR_LENGTH(CITY) AS name_length
  FROM
    STATION
  ORDER BY
    CHAR_LENGTH(CITY) ASC,   -- Shortest name
    CITY ASC
  LIMIT 1
)
UNION ALL
(
  SELECT
    CITY,
    CHAR_LENGTH(CITY) AS name_length
  FROM
    STATION
  ORDER BY
    CHAR_LENGTH(CITY) DESC,  -- Longest name
    CITY ASC
  LIMIT 1
);
```

---

## Question 3: City Revenue Analysis Through Multi-Table Joins

**Interview Context:**
"Our ride-sharing company operates in multiple cities. We have a complex data structure where cities contain users, and users take rides. Your task is to calculate the total revenue generated from each city, including cities that haven't generated any revenue yet. This is crucial for our expansion strategy and understanding market performance."

**Table Structures:**
```sql
Cities:
id | name
1  | New York
2  | Austin
3  | Dallas

Users:
id | city_id
1  | 1
2  | 2
3  | 1

Rides:
id | user_id | fare
1  | 1       | 20
2  | 1       | 30
3  | 2       | 10
4  | 3       | 15
```

**Expected Output:**
```
name     | earnings
Dallas   | 0
Austin   | 10
New York | 65
```

**Solution:**
```sql
SELECT
  c.name,
  COALESCE(SUM(r.fare), 0) AS earnings
FROM
  Cities AS c
LEFT JOIN
  Users AS u ON u.city_id = c.id
LEFT JOIN
  Rides AS r ON r.user_id = u.id
GROUP BY
  c.name
ORDER BY
  earnings ASC,
  c.name ASC;
```

---

## Question 4: Call Center Analytics

**Interview Context:**
"You're analyzing call center data to understand agent performance. We need to calculate the total number of calls made and total duration for each caller. However, our data has some quality issues - some records have NULL caller_ids that we need to exclude from our analysis."

**Table Structure:**
```sql
Calls:
caller_id | recipient_id | duration
1         | 2            | 60
1         | 3            | 40
2         | 1            | 20
NULL      | 3            | 10
```

**Expected Output:**
```
caller_id | total_calls | total_duration
1         | 2           | 100
2         | 1           | 20
```

**Solution:**
```sql
SELECT
    caller_id,
    COUNT(*) AS total_calls,
    SUM(duration) AS total_duration
FROM
    Calls
WHERE
    caller_id IS NOT NULL
GROUP BY
    caller_id
ORDER BY
    caller_id ASC;
```

---

## Question 5: Binary Tree Node Classification

**Interview Context:**
"We're storing a binary search tree in a database table. Each node has a value and points to its parent. Your job is to classify each node as either 'Root' (no parent), 'Leaf' (no children), or 'Inner' (has both parent and children). This is essential for our tree traversal algorithms."

**Table Structure:**
```sql
BST:
N (Node) | P (Parent)
1        | 2
2        | 5
3        | 2
5        | NULL
6        | 8
8        | 5
9        | 8
```

**Solution:**
```sql
SELECT 
    A.N,
    CASE 
        WHEN A.P IS NULL THEN 'Root'
        WHEN B.P IS NULL THEN 'Leaf'
        ELSE 'Inner'
    END AS NodeType
FROM BST A
LEFT JOIN BST B ON A.N = B.P
GROUP BY A.N, A.P
ORDER BY A.N;
```

---

## Question 6: Order Management - Non-Delivered Orders

**Interview Context:**
"Our logistics team needs to prioritize order fulfillment. They want to see the 5 earliest orders that haven't been delivered yet, so they can focus on the oldest pending orders first. This helps improve customer satisfaction by addressing delayed orders."

**Table Structure:**
```sql
orders:
id | order_date | status      | customer_id
1  | 2023-01-01 | PENDING     | 101
2  | 2023-01-02 | DELIVERED   | 102
3  | 2023-01-03 | SHIPPED     | 103
4  | 2023-01-04 | PENDING     | 104
5  | 2023-01-05 | CANCELLED   | 105
```

**Solution:**
```sql
SELECT 
    id, order_date, status, customer_id
FROM orders
WHERE status != 'DELIVERED'
ORDER BY order_date ASC, id ASC
LIMIT 5;
```

---

## Question 7: Product Wishlist Analysis

**Interview Context:**
"Our e-commerce platform has a wishlist feature. Marketing wants to understand which in-stock products are most desired by customers. Find the top 3 in-stock products that appear most frequently on wishlists, along with their prices and wishlist counts."

**Table Structures:**
```sql
products:
id | name      | price | in_stock
1  | Laptop    | 999   | 1
2  | Phone     | 699   | 1
3  | Tablet    | 399   | 0
4  | Watch     | 299   | 1

wishlists:
product_id | customer_email
1          | user1@email.com
1          | user2@email.com
2          | user1@email.com
4          | user3@email.com
```

**Solution:**
```sql
SELECT 
    p.name,
    p.price,
    COUNT(w.customer_email) AS total_wishes
FROM products p
JOIN wishlists w ON p.id = w.product_id
WHERE p.in_stock = 1
GROUP BY p.id, p.name, p.price
ORDER BY total_wishes DESC, p.name ASC
LIMIT 3;
```

---

## Question 8: Sales Performance Analysis

**Interview Context:**
"We're evaluating our sales representatives' performance. A rep is considered successful if they meet or exceed their monthly target for at least 7 months. This helps us identify top performers for promotion and recognize consistent achievers."

**Table Structures:**
```sql
sales_representatives:
id | email
1  | john@example.com
2  | jane@example.com

monthly_sales:
sales_representative_id | month   | amount
1                      | 2023-01 | 50000
1                      | 2023-02 | 45000

monthly_targets:
sales_representative_id | month   | amount
1                      | 2023-01 | 40000
1                      | 2023-02 | 50000
```

**Solution:**
```sql
SELECT 
    sr.email,
    COUNT(*) AS months_hit_target
FROM monthly_sales ms
JOIN monthly_targets mt 
    ON ms.sales_representative_id = mt.sales_representative_id
   AND ms.month = mt.month
JOIN sales_representatives sr 
    ON sr.id = ms.sales_representative_id
WHERE ms.amount >= mt.amount
GROUP BY sr.email
HAVING COUNT(*) >= 7
ORDER BY sr.email;
```

---

## Question 9: Finding Second Highest Salary

**Interview Context:**
"HR needs to understand salary distribution for compensation planning. They specifically want to know the second highest salary in the company. If there's no second highest salary (like when all employees have the same salary), return NULL."

**Table Structure:**
```sql
employees:
id | name    | salary
1  | Alice   | 80000
2  | Bob     | 90000
3  | Charlie | 85000
```

**Solution:**
```sql
SELECT 
    MAX(salary) AS second_highest_salary
FROM employees
WHERE salary < (
    SELECT MAX(salary) FROM employees
);
```

---

## Question 10: Nth Highest Salary (Generic Solution)

**Interview Context:**
"Building on the previous question, we need a more flexible solution. Create a query that can find the Nth highest distinct salary, where N can be any number. This will be useful for various HR analytics needs."

**Solution:**
```sql
SELECT 
    DISTINCT salary AS nth_highest_salary
FROM employees
ORDER BY salary DESC
LIMIT 1 OFFSET N - 1;  -- Replace N with the desired rank
```

---

## Question 11: Skills-Based Candidate Filtering

**Interview Context:**
"Our recruiting team is looking for candidates with specific technical skills. We need to find all candidates who have both Python and SQL skills listed in their profile. The skills are stored as comma-separated values in a single field."

**Table Structure:**
```sql
candidates:
id | name    | skills
1  | Alice   | Python,SQL,Java
2  | Bob     | JavaScript,HTML
3  | Charlie | Python,SQL,React
```

**Solution:**
```sql
SELECT 
    id, name
FROM candidates
WHERE skills LIKE '%Python%'
  AND skills LIKE '%SQL%';
```

---

## Question 12: Corporate Hierarchy Analysis

**Interview Context:**
"Amber's conglomerate just acquired several companies. Each company follows a strict hierarchy: Founder → Lead Manager → Senior Manager → Manager → Employee. We need to create a summary report showing the count of each role type per company. The data may contain duplicates, so be careful to count distinct values only."

**Table Structures:**
```sql
Company:
company_code | founder
C1          | Monika
C2          | Samantha

Lead_Manager:
lead_manager_code | company_code
LM1              | C1
LM2              | C2

Senior_Manager:
senior_manager_code | lead_manager_code | company_code
SM1                | LM1               | C1
SM2                | LM1               | C1
SM3                | LM2               | C2

Manager:
manager_code | senior_manager_code | lead_manager_code | company_code
M1          | SM1                 | LM1               | C1
M2          | SM3                 | LM2               | C2
M3          | SM3                 | LM2               | C2

Employee:
employee_code | manager_code | senior_manager_code | lead_manager_code | company_code
E1           | M1           | SM1                 | LM1               | C1
E2           | M1           | SM1                 | LM1               | C1
E3           | M2           | SM3                 | LM2               | C2
E4           | M3           | SM3                 | LM2               | C2
```

**Solution:**
```sql
SELECT
    c.company_code,                                    -- Select company code
    c.founder,                                         -- Select founder name
    COUNT(DISTINCT lm.lead_manager_code) AS lead_count,       -- Unique lead managers
    COUNT(DISTINCT sm.senior_manager_code) AS senior_count,   -- Unique senior managers
    COUNT(DISTINCT m.manager_code) AS manager_count,          -- Unique managers
    COUNT(DISTINCT e.employee_code) AS employee_count         -- Unique employees
FROM Company c
LEFT JOIN Lead_Manager lm
    ON c.company_code = lm.company_code               -- Join with Lead Manager table
LEFT JOIN Senior_Manager sm
    ON c.company_code = sm.company_code               -- Join with Senior Manager table
LEFT JOIN Manager m
    ON c.company_code = m.company_code                -- Join with Manager table
LEFT JOIN Employee e
    ON c.company_code = e.company_code                -- Join with Employee table
GROUP BY c.company_code, c.founder                    -- Group by to aggregate counts
ORDER BY c.company_code ASC;                          -- Sort alphabetically as required
```

---

## Question 13: Advanced Coupon Analytics with Window Functions

**Interview Context:**
"Our e-commerce platform uses various coupon codes to drive sales. The marketing team needs a comprehensive analysis including monthly sales per coupon, month-over-month growth rates, rolling averages, and ranking based on performance trends. This complex analysis will help optimize our promotional strategies."

**Table Structures:**
```sql
coupons:
id | coupon_code
1  | DISCOUNT50
2  | SAVE20
3  | COUPON123

orders:
coupon_id | dt                  | total_amount
1         | 2022-08-03 14:02:00 | 25.50
1         | 2022-08-15 16:40:00 | 54.22
1         | 2022-09-01 10:00:00 | 63.34
2         | 2022-09-02 09:30:00 | 33.00
3         | 2022-08-20 11:00:00 | 71.00
3         | 2022-09-10 13:15:00 | 41.00
3         | 2022-10-01 15:15:00 | 75.00
3         | 2022-10-11 17:45:00 | 25.00
```

**Solution:**
```sql
WITH cte AS (
    SELECT 
        c.coupon_code, 
        o.dt, 
        o.total_amount
    FROM coupons c
    JOIN orders o ON c.id = o.coupon_id
),

cte_2 AS (
    SELECT 
        coupon_code, 
        DATE_FORMAT(dt, '%Y-%m') AS month_year,
        SUM(total_amount) AS monthly_sales
    FROM cte
    GROUP BY coupon_code, DATE_FORMAT(dt, '%Y-%m')
),

cte_3 AS (
    SELECT 
        coupon_code, 
        month_year,
        monthly_sales,
        LAG(monthly_sales) OVER (
            PARTITION BY coupon_code 
            ORDER BY month_year
        ) AS prev_month_sales
    FROM cte_2
),

cte_4 AS (
    SELECT 
        coupon_code, 
        month_year,
        monthly_sales,
        CASE 
            WHEN prev_month_sales IS NULL OR prev_month_sales = 0 THEN 'N/A'
            ELSE ROUND(((monthly_sales - prev_month_sales) / prev_month_sales) * 100, 2)
        END AS mom_growth,
        ROUND(AVG(monthly_sales) OVER (
            PARTITION BY coupon_code 
            ORDER BY month_year 
            ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
        ), 2) AS avg_last_3_months
    FROM cte_3
)

SELECT 
    coupon_code,
    month_year,
    monthly_sales,
    mom_growth,
    RANK() OVER (
        PARTITION BY month_year 
        ORDER BY avg_last_3_months DESC
    ) AS usage_rank,
    avg_last_3_months
FROM cte_4
ORDER BY month_year ASC, usage_rank ASC;
```

---

## Question 14: Customer Segmentation by Basket Size

**Interview Context:**
"As a retail analyst, you need to segment customers based on their purchasing behavior. Calculate each customer's average basket size (total sales divided by number of transactions) and categorize them as High (>$30), Medium ($20-$30), or Low (<$20). This segmentation will drive our targeted marketing campaigns."

**Table Structures:**
```sql
wfm_transactions:
customer_id | store_id | transaction_date | transaction_id | product_id | sales
1          | 101      | 2017-01-15      | T001          | P001       | 25
1          | 101      | 2017-01-20      | T002          | P002       | 35
2          | 102      | 2017-02-10      | T003          | P001       | 15

wfm_stores:
store_id | store_brand | location
101      | WholeFoods  | NYC
102      | WholeFoods  | LA
```

**Solution:**
```sql
WITH customer_basket AS (
  SELECT 
    t.customer_id,
    s.store_brand,
    COUNT(DISTINCT t.transaction_id) AS transaction_count,
    SUM(t.sales) AS total_sales,
    SUM(t.sales) * 1.0 / COUNT(DISTINCT t.transaction_id) AS avg_basket_size,
    CASE 
      WHEN SUM(t.sales) * 1.0 / COUNT(DISTINCT t.transaction_id) > 30 THEN 'High'
      WHEN SUM(t.sales) * 1.0 / COUNT(DISTINCT t.transaction_id) BETWEEN 20 AND 30 THEN 'Medium'
      ELSE 'Low'
    END AS segment
  FROM wfm_transactions t
  JOIN wfm_stores s ON t.store_id = s.store_id
  WHERE YEAR(t.transaction_date) = 2017
  GROUP BY t.customer_id, s.store_brand
)

SELECT
  store_brand,
  segment,
  COUNT(*) AS number_customers,
  SUM(transaction_count) AS total_transactions,
  SUM(total_sales) AS total_sales,
  ROUND(SUM(total_sales) * 1.0 / NULLIF(SUM(transaction_count), 0), 2) AS avg_basket_size
FROM customer_basket
GROUP BY store_brand, segment
ORDER BY store_brand, segment;
```

---

## Key Concepts Covered

### 1. **Basic Filtering & Sorting**
- WHERE clauses with multiple conditions
- ORDER BY for result ordering
- Handling NULL values

### 2. **Aggregation Functions**
- COUNT, SUM, AVG, MAX, MIN
- GROUP BY for aggregation
- HAVING for filtered aggregation

### 3. **String Functions**
- CHAR_LENGTH for string length
- LIKE operator for pattern matching

### 4. **JOIN Operations**
- INNER JOIN for matching records
- LEFT JOIN for preserving left table records
- Multiple table joins

### 5. **Window Functions**
- RANK() and ROW_NUMBER()
- LAG() for previous row values
- Moving averages with ROWS BETWEEN

### 6. **Subqueries**
- Correlated subqueries
- EXISTS and NOT EXISTS
- Subqueries in SELECT clause

### 7. **CASE Statements**
- Conditional logic in SQL
- Multiple condition handling
- Creating categorical variables

### 8. **Date/Time Functions**
- DATE_FORMAT for date manipulation
- YEAR() function for filtering

### 9. **Advanced Analytics**
- Customer segmentation
- Cohort analysis concepts
- Performance ranking and comparison