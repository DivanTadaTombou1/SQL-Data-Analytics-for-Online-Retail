# SQL-Data-Analytics-for-Online-Retail

## Problem Statement

For my GitHub project, I aim to conduct an in-depth data analytics investigation on a fictional online retail database. My goal is to extract meaningful insights into customer behavior, product performance, and sales trends. This dataset consists of interconnected tables representing customers, products, orders, order items, and reviews.

## Project Goals

1. **Customer Analysis:**
   - I will analyze customer purchasing behavior by calculating total orders, total sales, and average order value per customer.
   - Segmentation of customers based on their purchasing frequency and order value will help me identify high-value segments.

2. **Product Performance Analysis:**
   - Evaluating the performance of products by analyzing total sales, average rating, total orders, and profitability is one of my priorities.
   - I intend to determine the top-selling products and their contribution to overall sales.

3. **Sales Trends Analysis:**
   - Analyzing monthly sales trends by region will help me identify patterns and seasonal fluctuations.
   - Calculating rolling averages and cumulative sales is crucial for understanding long-term trends and forecasting future performance.

4. **Predictive Modeling:**
   - I plan to employ advanced techniques such as linear regression and clustering analysis to predict future sales trends and identify potential growth opportunities.
   - Performing sales growth analysis for individual products will help me understand their market dynamics and potential for expansion.

5. **Customer Lifetime Value (CLV):**
   - Calculating customer lifetime value (CLV) will allow me to assess the long-term profitability of different customer segments.
   - Classifying customers into tiers based on their lifetime sales will help me tailor marketing strategies accordingly.

## Deliverables

1. I will provide comprehensive SQL queries incorporating advanced techniques such as window functions, subqueries, and derived metrics to perform detailed data analysis.
2. Optionally, I may create visualizations to illustrate key findings and insights derived from the analysis.
3. Documentation outlining my project methodology, SQL queries, and interpretation of results will be included.
4. Additionally, I may prepare a presentation summarizing the key findings and recommendations for stakeholders.

## Expected Outcome

By the end of this project, I aim to provide actionable insights to stakeholders, including marketing teams, product managers, and senior management. My objective is to optimize business strategies, improve customer engagement, and drive revenue growth in the online retail domain.

```
sql

WITH CustomerOrderStats AS (
    SELECT
        c.customer_id,
        c.name,
        c.region,
        COUNT(o.order_id) AS total_orders,
        SUM(oi.quantity * oi.item_price) AS total_sales,
        AVG(oi.quantity * oi.item_price) AS average_order_value,
        SUM(oi.quantity * p.cost) AS total_cost,
        DENSE_RANK() OVER (ORDER BY SUM(oi.quantity * oi.item_price) DESC) AS sales_rank,
        ROW_NUMBER() OVER (PARTITION BY c.region ORDER BY SUM(oi.quantity * oi.item_price) DESC) AS regional_rank
    FROM
        customers c
    JOIN
        orders o ON c.customer_id = o.customer_id
    JOIN
        order_items oi ON o.order_id = oi.order_id
    JOIN
        products p ON oi.product_id = p.product_id
    GROUP BY
        c.customer_id, c.name, c.region
),

ProductStats AS (
    SELECT
        p.product_id,
        p.product_name,
        p.category,
        SUM(oi.quantity * oi.item_price) AS total_sales,
        AVG(r.rating) AS average_rating,
        COUNT(DISTINCT o.order_id) AS total_orders,
        SUM(oi.quantity * p.cost) AS total_cost,
        SUM(oi.quantity * oi.item_price) - SUM(oi.quantity * p.cost) AS profit,
        STDDEV(oi.quantity * oi.item_price) AS sales_stddev
    FROM
        products p
    JOIN
        order_items oi ON p.product_id = oi.product_id
    JOIN
        orders o ON oi.order_id = o.order_id
    LEFT JOIN
        reviews r ON p.product_id = r.product_id
    GROUP BY
        p.product_id, p.product_name, p.category
),

MonthlySales AS (
    SELECT
        DATE_TRUNC('month', o.order_date) AS month,
        c.region,
        SUM(oi.quantity * oi.item_price) AS total_sales,
        SUM(oi.quantity * p.cost) AS total_cost,
        SUM(oi.quantity * oi.item_price) - SUM(oi.quantity * p.cost) AS profit
    FROM
        orders o
    JOIN
        customers c ON o.customer_id = c.customer_id
    JOIN
        order_items oi ON o.order_id = oi.order_id
    JOIN
        products p ON oi.product_id = p.product_id
    GROUP BY
        DATE_TRUNC('month', o.order_date), c.region
),

RollingMonthlySales AS (
    SELECT
        region,
        month,
        total_sales,
        AVG(total_sales) OVER (PARTITION BY region ORDER BY month ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS rolling_avg_sales,
        AVG(profit) OVER (PARTITION BY region ORDER BY month ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS rolling_avg_profit,
        SUM(total_sales) OVER (PARTITION BY region ORDER BY month ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS cumulative_sales
    FROM
        MonthlySales
),

CustomerLifetimeValue AS (
    SELECT
        customer_id,
        SUM(total_sales) AS lifetime_value,
        SUM(total_cost) AS lifetime_cost,
        SUM(total_sales) - SUM(total_cost) AS lifetime_profit,
        CASE
            WHEN SUM(total_sales) > 10000 THEN 'Platinum'
            WHEN SUM(total_sales) BETWEEN 5000 AND 10000 THEN 'Gold'
            WHEN SUM(total_sales) BETWEEN 1000 AND 5000 THEN 'Silver'
            ELSE 'Bronze'
        END AS customer_tier
    FROM
        CustomerOrderStats
    GROUP BY
        customer_id
),

CustomerSegments AS (
    SELECT
        c.customer_id,
        c.name,
        c.region,
        COALESCE(SUM(o.total_sales), 0) AS total_sales,
        COALESCE(SUM(o.total_orders), 0) AS total_orders,
        CASE
            WHEN COALESCE(SUM(o.total_sales), 0) > 1000 THEN 'High Value'
            ELSE 'Low Value'
        END AS value_segment,
        CASE
            WHEN COALESCE(SUM(o.total_orders), 0) > 10 THEN 'Frequent Buyer'
            ELSE 'Infrequent Buyer'
        END AS frequency_segment
    FROM
        customers c
    LEFT JOIN
        CustomerOrderStats o ON c.customer_id = o.customer_id
    GROUP BY
        c.customer_id, c.name, c.region
),

ProductPredictiveSales AS (
    SELECT
        product_id,
        product_name,
        category,
        month,
        total_sales,
        SUM(total_sales) OVER (PARTITION BY product_id ORDER BY month ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS rolling_total_sales,
        AVG(total_sales) OVER (PARTITION BY product_id ORDER BY month ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS rolling_avg_sales,
        ROW_NUMBER() OVER (PARTITION BY product_id ORDER BY month DESC) AS recent_sales_rank
    FROM (
        SELECT
            p.product_id,
            p.product_name,
            p.category,
            DATE_TRUNC('month', o.order_date) AS month,
            SUM(oi.quantity * oi.item_price) AS total_sales
        FROM
            products p
        JOIN
            order_items oi ON p.product_id = oi.product_id
        JOIN
            orders o ON oi.order_id = o.order_id
        GROUP BY
            p.product_id, p.product_name, p.category, DATE_TRUNC('month', o.order_date)
    ) subquery
),

ProductSalesTrend AS (
    SELECT
        product_id,
        product_name,
        category,
        month,
        total_sales,
        LAG(total_sales, 1) OVER (PARTITION BY product_id ORDER BY month) AS prev_month_sales,
        total_sales - LAG(total_sales, 1) OVER (PARTITION BY product_id ORDER BY month) AS sales_growth
    FROM
        ProductPredictiveSales
)

SELECT
    cs.customer_id,
    cs.name,
    cs.region,
    cs.total_orders,
    cs.total_sales,
    cs.average_order_value,
    cs.total_cost,
    cs.sales_rank,
    cs.regional_rank,
    clv.lifetime_value,
    clv.lifetime_profit,
    clv.customer_tier,
    ps.product_name,
    ps.category,
    ps.total_sales AS product_total_sales,
    ps.average_rating,
    ps.total_cost AS product_total_cost,
    ps.profit AS product_profit,
    ps.sales_stddev,
    ms.month,
    ms.total_sales AS monthly_total_sales,
    ms.profit AS monthly_profit,
    rms.rolling_avg_sales,
    rms.rolling_avg_profit,
    rms.cumulative_sales,
    seg.value_segment,
    seg.frequency_segment,
    pst.sales_growth
FROM
    CustomerOrderStats cs
LEFT JOIN
    ProductStats ps ON cs.customer_id = ps.product_id
LEFT JOIN
    MonthlySales ms ON cs.region = ms.region
LEFT JOIN
    RollingMonthlySales rms ON ms.region = rms.region AND ms.month = rms.month
LEFT JOIN
    CustomerLifetimeValue clv ON cs.customer_id = clv.customer_id
LEFT JOIN
    CustomerSegments seg ON cs.customer_id = seg.customer_id
LEFT JOIN
    ProductSalesTrend pst ON ps.product_id = pst.product_id AND pst.month = ms.month
ORDER BY
    cs.total_sales DESC, ps.product_total_sales DESC, ms.month ASC;

```
