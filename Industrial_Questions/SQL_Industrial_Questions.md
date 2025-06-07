# Industrial SQL Questions and Answers

This document contains real-world industrial SQL questions along with their solutions. These examples demonstrate how SQL is used to solve common business problems in various industries.

## Table of Contents
1. [E-commerce Analytics](#e-commerce-analytics)
2. [Financial Reporting](#financial-reporting)
3. [Supply Chain Management](#supply-chain-management)
4. [Healthcare Analytics](#healthcare-analytics)
5. [Customer Segmentation](#customer-segmentation)

## E-commerce Analytics

### Question 1: Customer Purchase Funnel Analysis
**Business Problem:** Analyze the customer journey from product view to purchase to identify drop-off points.

```sql
WITH funnel AS (
    SELECT
        COUNT(DISTINCT session_id) as total_sessions,
        COUNT(DISTINCT CASE WHEN event_type = 'product_view' THEN session_id END) as viewed_product,
        COUNT(DISTINCT CASE WHEN event_type = 'add_to_cart' THEN session_id END) as added_to_cart,
        COUNT(DISTINCT CASE WHEN event_type = 'checkout_started' THEN session_id END) as started_checkout,
        COUNT(DISTINCT CASE WHEN event_type = 'purchase_completed' THEN session_id END) as completed_purchase
    FROM user_events
    WHERE event_date = CURRENT_DATE - INTERVAL '7 days'
)
SELECT
    total_sessions,
    ROUND(viewed_product * 100.0 / total_sessions, 2) as view_rate,
    ROUND(added_to_cart * 100.0 / viewed_product, 2) as cart_add_rate,
    ROUND(started_checkout * 100.0 / added_to_cart, 2) as checkout_rate,
    ROUND(completed_purchase * 100.0 / started_checkout, 2) as conversion_rate
FROM funnel;
```

### Question 2: Product Recommendation Engine
**Business Problem:** Implement a "frequently bought together" recommendation system.

```sql
WITH product_pairs AS (
    SELECT
        oi1.product_id as product_1,
        oi2.product_id as product_2,
        COUNT(DISTINCT oi1.order_id) as times_bought_together
    FROM order_items oi1
    JOIN order_items oi2 
        ON oi1.order_id = oi2.order_id 
        AND oi1.product_id < oi2.product_id
    GROUP BY 1, 2
    HAVING COUNT(DISTINCT oi1.order_id) >= 10
    ORDER BY 3 DESC
    LIMIT 100
)
SELECT 
    p1.name as product_name,
    p2.name as recommended_product,
    pp.times_bought_together
FROM product_pairs pp
JOIN products p1 ON pp.product_1 = p1.id
JOIN products p2 ON pp.product_2 = p2.id;
```

## Financial Reporting

### Question 3: Monthly Revenue Growth
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
**Business Problem:** Calculate the 90-day customer lifetime value by acquisition channel.

```sql
WITH customer_metrics AS (
    SELECT
        u.acquisition_channel,
        COUNT(DISTINCT o.customer_id) as total_customers,
        COUNT(DISTINCT o.order_id) as total_orders,
        SUM(o.amount) as total_revenue,
        COUNT(DISTINCT o.order_id) * 1.0 / COUNT(DISTINCT o.customer_id) as avg_orders_per_customer,
        SUM(o.amount) / COUNT(DISTINCT o.customer_id) as avg_revenue_per_customer
    FROM users u
    JOIN orders o ON u.id = o.customer_id
    WHERE o.order_date BETWEEN CURRENT_DATE - 90 AND CURRENT_DATE
    GROUP BY 1
)
SELECT
    acquisition_channel,
    total_customers,
    total_orders,
    total_revenue,
    ROUND(avg_orders_per_customer, 2) as avg_orders_per_customer,
    ROUND(avg_revenue_per_customer, 2) as avg_revenue_per_customer,
    ROUND(avg_revenue_per_customer * 0.25, 2) as estimated_90day_clv -- Assuming 25% profit margin
FROM customer_metrics
ORDER BY avg_revenue_per_customer DESC;
```

## Supply Chain Management

### Question 5: Inventory Turnover Ratio
**Business Problem:** Calculate inventory turnover ratio by product category.

```sql
WITH monthly_inventory AS (
    SELECT
        p.category,
        DATE_TRUNC('month', i.date) as month,
        AVG(i.quantity_on_hand) as avg_inventory,
        SUM(oi.quantity) as units_sold
    FROM inventory i
    JOIN products p ON i.product_id = p.id
    LEFT JOIN order_items oi ON p.id = oi.product_id 
        AND oi.created_at BETWEEN DATE_TRUNC('month', i.date) AND DATE_TRUNC('month', i.date) + INTERVAL '1 month'
    WHERE i.date >= DATE_TRUNC('year', CURRENT_DATE)
    GROUP BY 1, 2
)
SELECT
    category,
    TO_CHAR(month, 'YYYY-MM') as month,
    ROUND(units_sold / NULLIF(avg_inventory, 0), 2) as inventory_turnover_ratio
FROM monthly_inventory
ORDER BY category, month;
```

### Question 6: Supplier Performance Analysis
**Business Problem:** Identify suppliers with late shipments.

```sql
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
