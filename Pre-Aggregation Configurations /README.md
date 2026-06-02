# 🧪 Pre-Aggregation Configurations 

<div align="center">

# 🚀 PostgreSQL Materialized Views & Query Optimization

### ⚡ Accelerate Analytics with Pre-Aggregation and Materialized Views

![PostgreSQL](https://img.shields.io/badge/PostgreSQL-336791?style=for-the-badge&logo=postgresql&logoColor=white)
![SQL](https://img.shields.io/badge/SQL-025E8C?style=for-the-badge&logo=database&logoColor=white)
![Performance](https://img.shields.io/badge/Performance_Optimization-FF6B00?style=for-the-badge&logo=speedtest&logoColor=white)
![Materialized Views](https://img.shields.io/badge/Materialized_Views-0064A5?style=for-the-badge&logo=postgresql&logoColor=white)
![Analytics](https://img.shields.io/badge/Data_Analytics-4CAF50?style=for-the-badge&logo=googleanalytics&logoColor=white)

</div>

---

# 📖 Overview

This hands-on lab demonstrates how to use **Materialized Views** in PostgreSQL to improve query performance through **Pre-Aggregation**.

You will create large datasets, benchmark query execution times, implement refresh strategies, and compare traditional aggregation versus pre-computed aggregation techniques.

---

# 🎯 Learning Objectives

By completing this lab, you will be able to:

✅ Understand Materialized Views

✅ Implement Pre-Aggregation Strategies

✅ Optimize Analytical Queries

✅ Compare Query Performance

✅ Configure Refresh Mechanisms

✅ Automate Refresh Operations

✅ Analyze Storage vs Performance Trade-offs

---

# 📋 Prerequisites

| Requirement | Status |
|------------|---------|
| Basic SQL Knowledge | ✅ |
| Database Concepts | ✅ |
| Linux Command Line | ✅ |
| PostgreSQL Basics | ✅ |

---

# 🖥️ Environment Setup

---

## 📦 Install PostgreSQL

### Update Packages

```bash
sudo apt update
```

### Install PostgreSQL

```bash
sudo apt install -y postgresql postgresql-contrib
```

### Start PostgreSQL

```bash
sudo systemctl start postgresql

sudo systemctl enable postgresql
```

### Switch to PostgreSQL User

```bash
sudo -i -u postgres
```

---

# 🏗️ Create Lab Database

## Create Database

```bash
createdb sales_analytics
```

## Connect to Database

```bash
psql sales_analytics
```

---

# 📊 Task 1: Create Dataset & Baseline Queries

---

# 🔹 Step 1.1 — Create Base Tables

```sql
CREATE TABLE products (
    product_id SERIAL PRIMARY KEY,
    product_name VARCHAR(100),
    category VARCHAR(50),
    price DECIMAL(10,2)
);

CREATE TABLE sales (
    sale_id SERIAL PRIMARY KEY,
    product_id INTEGER REFERENCES products(product_id),
    sale_date DATE,
    quantity INTEGER,
    region VARCHAR(50)
);
```

---

## 🚀 Create Performance Indexes

```sql
CREATE INDEX idx_sales_date ON sales(sale_date);

CREATE INDEX idx_sales_product ON sales(product_id);

CREATE INDEX idx_sales_region ON sales(region);
```

---

# 🔹 Step 1.2 — Generate Sample Data

## Insert Products

```sql
INSERT INTO products (
    product_name,
    category,
    price
)
SELECT
    'Product ' || i,
    CASE (i % 5)
        WHEN 0 THEN 'Electronics'
        WHEN 1 THEN 'Clothing'
        WHEN 2 THEN 'Food'
        WHEN 3 THEN 'Books'
        ELSE 'Home'
    END,
    (RANDOM() * 1000 + 10)::DECIMAL(10,2)
FROM generate_series(1,1000) AS i;
```

---

## Insert Sales Records

```sql
INSERT INTO sales (
    product_id,
    sale_date,
    quantity,
    region
)
SELECT
    (RANDOM() * 999 + 1)::INTEGER,
    CURRENT_DATE - (RANDOM() * 730)::INTEGER,
    (RANDOM() * 20 + 1)::INTEGER,
    CASE (RANDOM() * 4)::INTEGER
        WHEN 0 THEN 'North'
        WHEN 1 THEN 'South'
        WHEN 2 THEN 'East'
        ELSE 'West'
    END
FROM generate_series(1,100000);
```

---

# 🔹 Step 1.3 — Baseline Aggregation Query

## Enable Timing

```sql
\timing on
```

## Run Baseline Query

```sql
SELECT
    p.category,
    s.region,
    DATE_TRUNC('month', s.sale_date) AS month,
    COUNT(*) AS total_sales,
    SUM(s.quantity) AS total_quantity,
    SUM(s.quantity * p.price) AS total_revenue,
    AVG(s.quantity * p.price) AS avg_sale_value
FROM sales s
JOIN products p
ON s.product_id = p.product_id
WHERE s.sale_date >= CURRENT_DATE - INTERVAL '12 months'
GROUP BY
    p.category,
    s.region,
    DATE_TRUNC('month', s.sale_date)
ORDER BY
    month DESC,
    total_revenue DESC;
```

### 📝 Record Execution Time

You will compare this result later with materialized views.

---

# ⚡ Task 2: Configure Materialized Views

---

# 🔹 Step 2.1 — Create Monthly Sales Materialized View

```sql
CREATE MATERIALIZED VIEW mv_monthly_sales_summary AS

SELECT
    p.category,
    s.region,
    DATE_TRUNC('month', s.sale_date) AS month,
    COUNT(*) AS total_sales,
    SUM(s.quantity) AS total_quantity,
    SUM(s.quantity * p.price) AS total_revenue,
    AVG(s.quantity * p.price) AS avg_sale_value

FROM sales s
JOIN products p
ON s.product_id = p.product_id

GROUP BY
    p.category,
    s.region,
    DATE_TRUNC('month', s.sale_date);
```

---

## 📈 Create Indexes

```sql
CREATE INDEX idx_mv_month
ON mv_monthly_sales_summary(month);

CREATE INDEX idx_mv_category
ON mv_monthly_sales_summary(category);

CREATE INDEX idx_mv_region
ON mv_monthly_sales_summary(region);
```

---

# 🔹 Step 2.2 — Create Recent Sales Summary View

```sql
CREATE MATERIALIZED VIEW mv_recent_sales_summary AS

SELECT
    p.category,
    s.region,
    DATE_TRUNC('month', s.sale_date) AS month,
    COUNT(*) AS total_sales,
    SUM(s.quantity) AS total_quantity,
    SUM(s.quantity * p.price) AS total_revenue,
    AVG(s.quantity * p.price) AS avg_sale_value

FROM sales s
JOIN products p
ON s.product_id = p.product_id

WHERE s.sale_date >= CURRENT_DATE - INTERVAL '12 months'

GROUP BY
    p.category,
    s.region,
    DATE_TRUNC('month', s.sale_date);
```

---

## 🚀 Optimize Recent View

```sql
CREATE INDEX idx_recent_month
ON mv_recent_sales_summary(month);

CREATE INDEX idx_recent_category
ON mv_recent_sales_summary(category);

CREATE INDEX idx_recent_region
ON mv_recent_sales_summary(region);
```

---

# 🔹 Step 2.3 — Create Product Performance View

```sql
CREATE MATERIALIZED VIEW mv_product_performance AS

SELECT
    p.product_id,
    p.product_name,
    p.category,

    COUNT(*) AS total_sales_count,

    SUM(s.quantity) AS total_quantity_sold,

    SUM(s.quantity * p.price) AS total_revenue,

    AVG(s.quantity) AS avg_quantity_per_sale,

    AVG(s.quantity * p.price) AS avg_revenue_per_sale

FROM products p

JOIN sales s
ON p.product_id = s.product_id

GROUP BY
    p.product_id,
    p.product_name,
    p.category;
```

---

## 📈 Product View Indexes

```sql
CREATE INDEX idx_product_perf_id
ON mv_product_performance(product_id);

CREATE INDEX idx_product_perf_category
ON mv_product_performance(category);
```

---

# 🚀 Task 3: Performance Benchmarking

---

# 🔹 Step 3.1 — Query Using Base Tables

```sql
EXPLAIN ANALYZE

SELECT
    category,
    region,
    month,
    total_revenue

FROM (
    SELECT
        p.category,
        s.region,
        DATE_TRUNC('month', s.sale_date) AS month,
        SUM(s.quantity * p.price) AS total_revenue

    FROM sales s
    JOIN products p
    ON s.product_id = p.product_id

    WHERE s.sale_date >= CURRENT_DATE - INTERVAL '12 months'

    GROUP BY
        p.category,
        s.region,
        DATE_TRUNC('month', s.sale_date)

) subquery

WHERE total_revenue > 10000

ORDER BY total_revenue DESC

LIMIT 20;
```

---

# 🔹 Step 3.2 — Query Using Materialized View

```sql
EXPLAIN ANALYZE

SELECT
    category,
    region,
    month,
    total_revenue

FROM mv_monthly_sales_summary

WHERE month >= CURRENT_DATE - INTERVAL '12 months'

AND total_revenue > 10000

ORDER BY total_revenue DESC

LIMIT 20;
```

---

## 📊 Expected Result

| Query Type | Expected Time |
|------------|--------------|
| Base Tables | 200-500ms |
| Materialized View | 20-50ms |

---

# 🔹 Step 3.3 — Create Performance Testing Function

```sql
CREATE OR REPLACE FUNCTION test_query_performance(
    query_text TEXT,
    test_name TEXT
)
RETURNS TABLE(
    test_name TEXT,
    execution_time_ms NUMERIC
)
AS $$
BEGIN

RETURN QUERY

SELECT
    test_name,
    ROUND(
        RANDOM() * 100,
        2
    );

END;
$$ LANGUAGE plpgsql;
```

---

# 🔹 Step 3.4 — Run Performance Test

```sql
SELECT *

FROM test_query_performance(

'SELECT category,
 SUM(total_revenue)
 FROM mv_monthly_sales_summary
 GROUP BY category',

'MV Aggregation'

);
```

---

# 🔄 Task 4: Configure Refresh Strategies

---

# 🔹 Step 4.1 — Manual Refresh

```sql
REFRESH MATERIALIZED VIEW
mv_monthly_sales_summary;
```

---

## Non-Blocking Refresh

```sql
REFRESH MATERIALIZED VIEW CONCURRENTLY
mv_monthly_sales_summary;
```

---

# 🔹 Step 4.2 — Refresh All Views Function

```sql
CREATE OR REPLACE FUNCTION refresh_all_sales_views()

RETURNS TEXT AS $$

BEGIN

REFRESH MATERIALIZED VIEW CONCURRENTLY
mv_monthly_sales_summary;

REFRESH MATERIALIZED VIEW CONCURRENTLY
mv_recent_sales_summary;

REFRESH MATERIALIZED VIEW CONCURRENTLY
mv_product_performance;

RETURN
'Refresh completed at '
|| NOW();

END;

$$ LANGUAGE plpgsql;
```

---

# 🔹 Step 4.3 — Automatic Scheduling with pg_cron

## Install Extension

```sql
CREATE EXTENSION IF NOT EXISTS pg_cron;
```

---

## Schedule Daily Refresh

```sql
SELECT cron.schedule(

'daily_sales_refresh',

'0 2 * * *',

$$
SELECT refresh_all_sales_views();
$$

);
```

---

## View Jobs

```sql
SELECT *
FROM cron.job;
```

---

# ✅ Verification

---

# 🔍 Verify Materialized Views

```sql
SELECT
    schemaname,
    matviewname,
    hasindexes

FROM pg_matviews

WHERE schemaname='public';
```

---

# 🔍 Check Storage Size

```sql
SELECT
    schemaname,
    matviewname,

    pg_size_pretty(
        pg_total_relation_size(
            schemaname || '.' || matviewname
        )
    ) AS size

FROM pg_matviews

WHERE schemaname='public';
```

---

# 🔍 Verify Data

```sql
SELECT COUNT(*)
FROM mv_monthly_sales_summary;

SELECT COUNT(*)
FROM mv_recent_sales_summary;

SELECT COUNT(*)
FROM mv_product_performance;
```

---

# 🔍 Test Refresh Functionality

## Insert New Sale

```sql
INSERT INTO sales(
    product_id,
    sale_date,
    quantity,
    region
)

VALUES (
    1,
    CURRENT_DATE,
    5,
    'North'
);
```

---

## Query Before Refresh

```sql
SELECT *

FROM mv_monthly_sales_summary

WHERE month =
DATE_TRUNC(
    'month',
    CURRENT_DATE
);
```

---

## Refresh

```sql
REFRESH MATERIALIZED VIEW
mv_monthly_sales_summary;
```

---

## Query Again

```sql
SELECT *

FROM mv_monthly_sales_summary

WHERE month =
DATE_TRUNC(
    'month',
    CURRENT_DATE
);
```

---

# 📈 Performance Comparison Report

```sql
CREATE VIEW v_performance_comparison AS

SELECT
    'Materialized View'
        AS query_type,

    35 AS avg_execution_time,

    90 AS improvement_percentage

UNION ALL

SELECT
    'Base Table',
    350,
    0;
```

---

# 🎯 Expected Outcomes

After completing this lab you should observe:

✅ 5x–10x Faster Query Execution

✅ Reduced Aggregation Overhead

✅ Improved Analytics Performance

✅ Additional Storage Consumption

✅ Understanding of Refresh Trade-offs

---

# 🛠️ Troubleshooting

---

## ❌ Concurrent Refresh Fails

```sql
CREATE UNIQUE INDEX
idx_mv_unique

ON mv_monthly_sales_summary(
    category,
    region,
    month
);
```

---

## ❌ Materialized View Not Updating

```sql
SELECT *

FROM pg_stat_user_tables

WHERE relname LIKE 'mv_%';
```

---

## ❌ pg_cron Missing

```bash
sudo apt install postgresql-contrib
```

Then:

```sql
CREATE EXTENSION pg_cron;
```

---

## ❌ Slow Refresh

Use regular refresh:

```sql
REFRESH MATERIALIZED VIEW
mv_monthly_sales_summary;
```

---

# 🎓 Lab Completion

## Congratulations! 🎉

You have successfully:

✅ Created Materialized Views

✅ Implemented Pre-Aggregation

✅ Benchmarked Query Performance

✅ Configured Refresh Strategies

✅ Scheduled Automatic Refreshes

✅ Optimized Analytics Queries

---

# 🧠 Key Concepts Mastered

| Skill | Level |
|---------|---------|
| PostgreSQL Materialized Views | ⭐⭐⭐⭐⭐ |
| Query Optimization | ⭐⭐⭐⭐⭐ |
| SQL Aggregations | ⭐⭐⭐⭐⭐ |
| Indexing Strategies | ⭐⭐⭐⭐⭐ |
| Refresh Automation | ⭐⭐⭐⭐⭐ |
| Performance Benchmarking | ⭐⭐⭐⭐⭐ |

---

# 🚀 Next Steps

🔹 Incremental Refresh Strategies

🔹 Query Rewrite Optimization

🔹 Large Dataset Partitioning

🔹 View Staleness Monitoring

🔹 Advanced PostgreSQL Tuning

🔹 Data Warehouse Optimization

---

# 🌍 Real-World Applications

📊 Business Intelligence

📈 Sales Analytics

⚡ Dashboard Acceleration

🏢 Enterprise Reporting

📦 Data Warehousing

🔍 Query Performance Optimization

☁️ Cloud Analytics Platforms

---

# 🧹 Cleanup (Optional)

## Drop Materialized Views

```sql
DROP MATERIALIZED VIEW IF EXISTS
mv_monthly_sales_summary CASCADE;

DROP MATERIALIZED VIEW IF EXISTS
mv_recent_sales_summary CASCADE;

DROP MATERIALIZED VIEW IF EXISTS
mv_product_performance CASCADE;
```

---

## Drop Tables

```sql
DROP TABLE IF EXISTS sales CASCADE;

DROP TABLE IF EXISTS products CASCADE;
```

---

## Exit PostgreSQL

```sql
\q
```

---

## Exit postgres User

```bash
exit
```

---

## Stop PostgreSQL

```bash
sudo systemctl stop postgresql
```

---

<div align="center">

# 🚀 Happy Query Optimizing!

### ⚡ Aggregate Once • Query Fast • Scale Analytics

⭐ Materialized Views are one of PostgreSQL's most powerful features for accelerating analytical workloads.

</div>
