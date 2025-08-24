# SQL Data Analytics Project

In the previous step ( [sql-eda-project](https://github.com/devoodian/sql-eda-project) ) our goal with EDA was to explore the data and uncover initial insights. Now, we move to the next, more advanced phase, where we perform deeper analyses, such as tracking trends over time, evaluating performance, and segmenting customers and products. The ultimate objective is to address two real business requirements and generate two high-quality SQL reports.

🔍 Want to see the SQL Data Warehouse project this Analysis is based on? Check it out here: [sql-dwh-project](https://github.com/devoodian/sql-dwh-project)

## 1. Change Over Time Analysis

**Goal**: Track trends and seasonality in sales and customer activity over time.

- **Query**: Analyse sales performance over time

```sql
SELECT 
    DATETRUNC(month, order_date) AS order_date, 
    SUM(sales_amount) AS total_sales, 
    COUNT(DISTINCT customer_key) AS total_customers, 
    SUM(quantity) AS total_quantity
FROM gold.fact_sales
WHERE order_date IS NOT NULL
GROUP BY DATETRUNC(month, order_date)
ORDER BY DATETRUNC(month, order_date);
```
📌 **Insight**:
- Sales grew steadily through 2011--2012, indicating consistent market expansion.

- From early 2013, sales and customers surged sharply, marking a clear scale-up phase and accelerated market penetration.

- Year-end months (November--December) show recurring peaks, suggesting strong seasonal demand patterns (likely linked to holiday cycles).


## 2. Cumulative Analysis

**Goal**: Track cumulative performance and moving averages to identify long-term growth trends.

- **Query**: Calculate the total sales per year with cumulative totals and moving averages

```sql
SELECT
    order_date,
    total_sales,
    SUM(total_sales) OVER (ORDER BY order_date) AS running_total_sales,
    AVG(avg_price) OVER (ORDER BY order_date) AS moving_average_price
FROM
(
    SELECT 
        DATETRUNC(year, order_date) AS order_date,
        SUM(sales_amount) AS total_sales,
        AVG(price) AS avg_price
    FROM gold.fact_sales
    WHERE order_date IS NOT NULL
    GROUP BY DATETRUNC(year, order_date)
) t
```
📌 **Insight**:
- Cumulative sales reached nearly **29.3M** by the end of 2013, showing strong long-term growth.

- 2013 contributed the largest yearly sales (~16.3M), accounting for more than half of the total accumulated revenue.

- The moving average price shows a **consistent decline** from ~3100 in 2010 to ~1668 in 2014, suggesting pricing strategies or product mix shifts made items more accessible over time.

## 3. Performance Analysis

**Goal**: Benchmark product performance against averages and track year-over-year growth or decline.

- **Query**: Compare yearly product sales to average and previous year performance

```sql
WITH yearly_product_sales AS (
    SELECT
        YEAR(f.order_date) AS order_year,
        p.product_name,
        SUM(f.sales_amount) AS current_sales
    FROM gold.fact_sales f
    LEFT JOIN gold.dim_products p
        ON f.product_key = p.product_key
    WHERE f.order_date IS NOT NULL
    GROUP BY 
        YEAR(f.order_date),
        p.product_name
)
SELECT
    order_year,
    product_name,
    current_sales,
    AVG(current_sales) OVER (PARTITION BY product_name) AS avg_sales,
    current_sales - AVG(current_sales) OVER (PARTITION BY product_name) AS diff_avg,
    CASE 
        WHEN current_sales - AVG(current_sales) OVER (PARTITION BY product_name) > 0 THEN 'Above Avg'
        WHEN current_sales - AVG(current_sales) OVER (PARTITION BY product_name) < 0 THEN 'Below Avg'
        ELSE 'Avg'
    END AS avg_change,
    -- Year-over-Year Analysis
    LAG(current_sales) OVER (PARTITION BY product_name ORDER BY order_year) AS py_sales,
    current_sales - LAG(current_sales) OVER (PARTITION BY product_name ORDER BY order_year) AS diff_py,
    CASE 
        WHEN current_sales - LAG(current_sales) OVER (PARTITION BY product_name ORDER BY order_year) > 0 THEN 'Increase'
        WHEN current_sales - LAG(current_sales) OVER (PARTITION BY product_name ORDER BY order_year) < 0 THEN 'Decrease'
        ELSE 'No Change'
    END AS py_change
FROM yearly_product_sales
ORDER BY product_name, order_year;
```
📌 **Insight**:
- Most products exhibit a **surge in 2013**, outperforming both their historical averages and prior-year sales, reflecting a breakout growth phase.

- However, **2014 shows sharp declines** across many SKUs, dropping below average and losing year-over-year momentum, which may indicate **market saturation, inventory constraints, or declining demand**.

- Seasonal or promotional products (e.g., apparel, helmets) peak strongly one year but collapse the next, highlighting **short product lifecycles and dependency on marketing pushes**.

- Core products like **high-end bikes and tires** dominate revenue growth during expansion years, reinforcing their role as **key revenue drivers**.

## 4. Data Segmentation Analysis

**Goal**: Group data into meaningful categories to uncover product, customer, or regional patterns for targeted insights.

- **Query 1**: Segment products into cost ranges and count how many products fall into each segment

```sql
WITH product_segments AS (
    SELECT
        product_key,
        product_name,
        cost,
        CASE 
            WHEN cost < 100 THEN 'Below 100'
            WHEN cost BETWEEN 100 AND 500 THEN '100-500'
            WHEN cost BETWEEN 500 AND 1000 THEN '500-1000'
            ELSE 'Above 1000'
        END AS cost_range
    FROM gold.dim_products
)
SELECT 
    cost_range,
    COUNT(product_key) AS total_products
FROM product_segments
GROUP BY cost_range
ORDER BY total_products DESC;
```
📌 **Insight**:
- The majority of products are **low-cost (Below €100, 110 items)** or **mid-range (€100--500, 101 items)**, showing strong affordability coverage.

- Only **39 premium products (> €1000)** exist, suggesting a **niche high-value segment** that could be leveraged for upselling or luxury branding.

**Query 2**: Segment customers based on spending behavior and lifecycle

```sql
WITH customer_spending AS (
    SELECT
        c.customer_key,
        SUM(f.sales_amount) AS total_spending,
        MIN(order_date) AS first_order,
        MAX(order_date) AS last_order,
        DATEDIFF(month, MIN(order_date), MAX(order_date)) AS lifespan
    FROM gold.fact_sales f
    LEFT JOIN gold.dim_customers c
        ON f.customer_key = c.customer_key
    GROUP BY c.customer_key
)
SELECT 
    customer_segment,
    COUNT(customer_key) AS total_customers
FROM (
    SELECT 
        customer_key,
        CASE 
            WHEN lifespan >= 12 AND total_spending > 5000 THEN 'VIP'
            WHEN lifespan >= 12 AND total_spending <= 5000 THEN 'Regular'
            ELSE 'New'
        END AS customer_segment
    FROM customer_spending
) AS segmented_customers
GROUP BY customer_segment
ORDER BY total_customers DESC;
```
📌 **Insight**:
- A significant majority of customers are **New (14,631)**, reflecting **high acquisition but limited retention**.

- Only **1,655 customers qualify as VIPs**, yet they represent the **most valuable long-term segment** worth prioritizing with loyalty programs.

- **Regular customers (2,198)** indicate stable but **low-spending retention**, suggesting room for **upselling campaigns** to push them toward VIP status.

## 5. Part-to-Whole Analysis

**Goal**: Compare performance across categories to understand relative contribution and dominance within overall sales.

- **Query**: Identify which categories contribute the most to overall sales

```sql
WITH category_sales AS (
    SELECT
        p.category,
        SUM(f.sales_amount) AS total_sales
    FROM gold.fact_sales f
    LEFT JOIN gold.dim_products p
        ON p.product_key = f.product_key
    GROUP BY p.category
)
SELECT
    category,
    total_sales,
    SUM(total_sales) OVER () AS overall_sales,
    ROUND((CAST(total_sales AS FLOAT) / SUM(total_sales) OVER ()) * 100, 2) AS percentage_of_total
FROM category_sales
ORDER BY total_sales DESC;
```
📌 **Insight**:
- **Bikes dominate sales**, contributing **~96.5% of total revenue (€28.3M)**, making them the **core driver of business performance**.

- **Accessories (2.4%)** and **Clothing (1.2%)** play a **minor role**, yet still present **cross-selling opportunities** to complement bike purchases.

- Heavy dependence on a single category (**Bikes**) highlights a **concentration risk**; diversifying into higher-margin accessories or apparel could reduce vulnerability and increase overall profitability.

## Reports
Here you have access to two key reports for better data analysis. The **Customer Report** examines customer behavior and characteristics, including key metrics such as number of orders, average monthly spending, and date of last purchase. The **Product Report** analyzes product performance, including segmentation by sales, number of purchases, and other financial metrics. To access these reports, please use the links below:

- **[Customer Report](https://github.com/devoodian/sql-data-analytics-project/blob/main/datasets/gold.report_customers.csv)**

- **[Product Report](https://github.com/devoodian/sql-data-analytics-project/blob/main/datasets/gold.report_products.csv)**
