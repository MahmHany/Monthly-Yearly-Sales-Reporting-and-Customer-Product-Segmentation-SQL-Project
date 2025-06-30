
# 📈 Monthly & Yearly Sales Reporting and Customer/Product Segmentation — SQL Project

This SQL-based project is a comprehensive sales analytics and segmentation solution built using structured queries over a star-schema data warehouse. It covers monthly trends, customer lifecycle metrics, product performance classification, and summarized analytical views using SQL Server (compatible with Snowflake/PostgreSQL with minor syntax adjustments).

---

## 📌 Project Objectives

- Track monthly and yearly sales performance by customer count, quantity, and revenue
- Calculate monthly and yearly running totals
- Analyze product sales performance including year-over-year and average-based performance
- Identify top-performing product categories contributing to overall revenue
- Segment products based on cost
- Segment customers based on total spending and engagement duration (lifespan)
- Create reusable views for customer and product reporting

---

## 📁 Data Sources

- `fact_sales`: Transactional fact table with columns like `order_date`, `sales_amount`, `quantity`, `customer_key`, `product_key`, etc.
- `dim_customers`: Dimension table with customer profile information
- `dim_products`: Product metadata including cost, category, and subcategory

---

## 🔍 Key Features & SQL Queries

### 📅 1. Monthly & Yearly Sales Trend Analysis

```sql
SELECT  
    YEAR(order_date) AS order_year,
    MONTH(order_date) AS order_month,
    SUM(sales_amount) AS total_sales,
    COUNT(DISTINCT customer_key) AS total_customer,
    SUM(quantity) AS total_quantity
FROM gold.fact_sales
WHERE order_date IS NOT NULL
GROUP BY YEAR(order_date), MONTH(order_date)
ORDER BY YEAR(order_date), MONTH(order_date);
```

Tracks monthly sales and engagement. Measures total unique customers and quantity sold.

Alternate version (for formatting & sorting by volume):

```sql
SELECT  
    FORMAT(order_date, 'yyyy-MMM') AS order_month,
    SUM(sales_amount) AS total_sales,
    COUNT(DISTINCT customer_key) AS total_customer,
    SUM(quantity) AS total_quantity
FROM gold.fact_sales
WHERE order_date IS NOT NULL
GROUP BY FORMAT(order_date, 'yyyy-MMM')
ORDER BY total_sales DESC;
```

### 🔄 2. Running Total Calculation (Monthly Within Each Year)

```sql
SELECT 
    order_month,
    total_sales,
    SUM(total_sales) OVER (PARTITION BY YEAR(order_month) ORDER BY order_month) AS running_total
FROM (
    SELECT 
        CAST(DATEFROMPARTS(YEAR(order_date), MONTH(order_date), 1) AS DATE) AS order_month,
        SUM(sales_amount) AS total_sales
    FROM gold.fact_sales
    WHERE order_date IS NOT NULL
    GROUP BY YEAR(order_date), MONTH(order_date)
) AS t;
```

Calculates cumulative sales over time per year (helpful for pacing analysis).

### 🧮 3. Product Performance Evaluation (YoY & Avg-Based)

```sql
WITH yearly_product_sales AS (
    SELECT 
        YEAR(order_date) AS order_year,
        p.product_name,
        SUM(sales_amount) AS current_sales
    FROM gold.fact_sales f
    LEFT JOIN gold.dim_products p ON f.product_key = p.product_key
    WHERE order_date IS NOT NULL
    GROUP BY YEAR(order_date), p.product_name
)
SELECT 
    order_year,
    product_name,
    current_sales,
    AVG(current_sales) OVER(PARTITION BY product_name) AS avg_sales,
    current_sales - AVG(current_sales) OVER(PARTITION BY product_name) AS avg_diff,
    CASE 
        WHEN current_sales > AVG(...) THEN 'Above Avg'
        WHEN current_sales < AVG(...) THEN 'Below Avg'
        ELSE 'Average'
    END AS avg_change,
    LAG(current_sales) OVER (...) AS py_sales,
    current_sales - LAG(...) AS py_diff,
    CASE ... END AS py_change
FROM yearly_product_sales;
```

Classifies products yearly as Above Avg / Below Avg / Average. Performs YoY comparisons to tag sales as Increasing / Decreasing / No Change.

### 📊 4. Category Contribution to Overall Sales

```sql
WITH category_sales AS (
    SELECT category, SUM(sales_amount) AS total_sales
    FROM gold.fact_sales f
    LEFT JOIN gold.dim_products p ON f.product_key = p.product_key
    GROUP BY category
)
SELECT 
    category,
    total_sales,
    SUM(total_sales) OVER() AS overall_sales,
    CONCAT(ROUND(CAST(total_sales AS FLOAT) / SUM(...) * 100, 2), '%') AS percentage_of_total
FROM category_sales
ORDER BY total_sales DESC;
```

Helps identify which product categories drive the most revenue.

### 💰 5. Product Segmentation by Cost Range

```sql
WITH product_segments AS (
    SELECT 
        product_key,
        cost,
        CASE 
            WHEN cost < 100 THEN 'Below 100'
            WHEN cost BETWEEN 100 AND 500 THEN '100-500'
            ...
        END AS cost_range
    FROM gold.dim_products
)
SELECT cost_range, COUNT(*) AS total_products
FROM product_segments
GROUP BY cost_range;
```

Breaks down product portfolio into cost-based segments.

### 👥 6. Customer Segmentation by Spending & Lifespan

```sql
WITH customer_spending AS (
    SELECT 
        customer_key,
        SUM(sales_amount) AS total_spending,
        DATEDIFF(MONTH, MIN(order_date), MAX(order_date)) AS lifespan
    FROM gold.fact_sales f
    LEFT JOIN gold.dim_customers c ON f.customer_key = c.customer_key
    GROUP BY c.customer_key
),
customer_segmented AS (
    SELECT 
        customer_key,
        total_spending,
        lifespan,
        CASE 
            WHEN lifespan >= 12 AND total_spending > 5000 THEN 'VIP'
            WHEN lifespan >= 12 THEN 'Regular'
            ELSE 'New'
        END AS customer_segment
)
SELECT customer_segment, COUNT(*) AS total_customers
FROM customer_segmented
GROUP BY customer_segment
ORDER BY total_customers DESC;
```

Categorizes customers into VIP, Regular, and New based on spending behavior and engagement duration.

### 📋 7. report_customer View — Full Customer Profile

```sql
CREATE VIEW gold.report_customer AS
-- Joins fact & dimension, calculates age, behavior metrics, and segments
```

Includes:
- Customer age group (20-29, 30-39, etc.)
- Last order date, recency
- Total orders, products, quantity, and sales
- Customer segments
- Average monthly spend and average order value

### 📦 8. report_products View — Full Product Profile

```sql
CREATE VIEW gold.report_products AS
-- Joins product details, aggregates sales metrics, and classifies product performance
```

Includes:
- Last sale date & recency
- Product performance segmentation (High-Performer, Mid-Range, Low-Performer)
- Average selling price
- Average order revenue and monthly revenue

---

## 📊 Sample Insights Enabled

- Monthly growth tracking with running totals
- Customer segmentation for loyalty or retention strategies
- Product performance tagging for lifecycle and marketing decisions
- Revenue concentration by category for strategic planning
- Automated customer/product reports via SQL views for easy dashboard integration

---

## 💻 Tech Stack

- SQL Server or Snowflake/PostgreSQL
- SSMS / DBeaver for query execution
- Power BI / Tableau (optional for visualization layer)

---

## 🧠 Why This Project?

This project simulates how a data analyst or BI developer would explore transactional data, generate insights, and build operationally ready reporting layers. It’s ideal for:
- Building analytical dashboards
- Supporting business decision-making
- Designing data marts or reporting views
