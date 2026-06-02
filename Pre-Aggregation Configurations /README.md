# 🧪 Lab: Pre-Aggregation Configurations

This lab covers the implementation, optimization, and management of pre-aggregated data structures in PostgreSQL using Materialized Views.

## 📋 Prerequisites

* Basic SQL knowledge (`SELECT`, `JOIN`, `WHERE` clauses)
* Understanding of database concepts (tables, indexes)
* Familiarity with the Linux command line
* PostgreSQL basics (connecting, running queries)

## 🎯 Learning Objectives

By completing this lab, you will:
* 💡 Understand materialized views and their performance benefits.
* ⚙️ Configure materialized views for query optimization.
* ⏱️ Measure and compare query performance with and without pre-aggregation.
* 🔄 Implement refresh strategies for materialized views.

---

## 🛠️ Environment Setup

### 1. 📥 Install PostgreSQL

```bash
# Update package list
sudo apt update

# Install PostgreSQL
sudo apt install -y postgresql postgresql-contrib

# Start PostgreSQL service
sudo systemctl start postgresql
sudo systemctl enable postgresql

# Switch to postgres user
sudo -i -u postgres
```

### 2. 🗄️ Create Lab Database

```bash
# Create database
createdb sales_analytics

# Connect to database
psql sales_analytics
```

---

## 📊 Task 1: Create Sample Dataset and Baseline Queries

### Step 1: 🧱 Create Base Tables

Execute the following SQL to create the sample schema and base indexes:

```sql
-- Create products table
CREATE TABLE products (
    product_id SERIAL PRIMARY KEY,
    product_name VARCHAR(100),
    category VARCHAR(50),
    price DECIMAL(10,2)
);

-- Create sales table
CREATE TABLE sales (
    sale_id SERIAL PRIMARY KEY,
    product_id INTEGER REFERENCES products(product_id),
    sale_date DATE,
    quantity INTEGER,
    region VARCHAR(50)
);

-- Create indexes for better performance
CREATE INDEX idx_sales_date ON sales(sale_date);
CREATE INDEX idx_sales_product ON sales(product_id);
CREATE INDEX idx_sales_region ON sales(region);
```

### Step 2: 🎲 Generate Sample Data

Populate the tables with mock data (1,000 products and 100,000 sales records):

```sql
-- Insert sample products
INSERT INTO products (product_name, category, price)
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
FROM generate_series(1, 1000) AS i;

-- Insert sample sales (100,000 records)
INSERT INTO sales (product_id, sale_date, quantity, region)
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
FROM generate_series(1, 100000);
```

### Step 3: 📉 Create Complex Aggregation Query

Run the baseline query on the raw tables and note the execution time.

```sql
-- Enable timing to measure query performance
\timing on

-- Complex aggregation query (baseline)
SELECT 
    p.category,
    s.region,
    DATE_TRUNC('month', s.sale_date) AS month,
    COUNT(*) AS total_sales,
    SUM(s.quantity) AS total_quantity,
    SUM(s.quantity * p.price) AS total_revenue,
    AVG(s.quantity * p.price) AS avg_sale_value
FROM sales s
JOIN products p ON s.product_id = p.product_id
WHERE s.sale_date >= CURRENT_DATE - INTERVAL '12 months'
GROUP BY p.category, s.region, DATE_TRUNC('month', s.sale_date)
ORDER BY month DESC, total_revenue DESC;
```

---

## 🏗️ Task 2: Configure Materialized Views

### Step 1: 🧩 Create Basic Materialized View

```sql
-- Create materialized view for monthly sales summary
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
JOIN products p ON s.product_id = p.product_id
GROUP BY p.category, s.region, DATE_TRUNC('month', s.sale_date);

-- Create index on materialized view
CREATE INDEX idx_mv_month ON mv_monthly_sales_summary(month);
CREATE INDEX idx_mv_category ON mv_monthly_sales_summary(category);
CREATE INDEX idx_mv_region ON mv_monthly_sales_summary(region);
```

### Step 2: 🔍 Create Advanced Materialized View with Filters

**🚀 Implementation:** Create a materialized view for recent sales filtering for the last 12 months.

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
JOIN products p ON s.product_id = p.product_id
WHERE s.sale_date >= CURRENT_DATE - INTERVAL '12 months'
GROUP BY p.category, s.region, DATE_TRUNC('month', s.sale_date);

-- Indexes to optimize filter and sort patterns
CREATE INDEX idx_mv_recent_month ON mv_recent_sales_summary(month);
CREATE INDEX idx_mv_recent_cat_reg ON mv_recent_sales_summary(category, region);
```

### Step 3: 💎 Create Product Performance View

**🚀 Implementation:** Create a materialized view for evaluating metrics grouped per product.

```sql
CREATE MATERIALIZED VIEW mv_product_performance AS
SELECT 
    p.product_id,
    p.product_name,
    p.category,
    COUNT(s.sale_id) AS total_sales_count,
    SUM(s.quantity) AS total_quantity_sold,
    SUM(s.quantity * p.price) AS total_revenue,
    AVG(s.quantity) AS avg_quantity_per_sale,
    AVG(s.quantity * p.price) AS avg_revenue_per_sale
FROM products p
LEFT JOIN sales s ON p.product_id = s.product_id
GROUP BY p.product_id, p.product_name, p.category;

-- Unique index required for subsequent concurrent refreshes
CREATE UNIQUE INDEX idx_mv_prod_perf_id ON mv_product_performance(product_id);
CREATE INDEX idx_mv_prod_perf_cat ON mv_product_performance(category);
```

---

## ⚡ Task 3: Test Aggregation Speed

### Step 1: ⚖️ Compare Query Performance

Run both blocks and view the `EXPLAIN ANALYZE` metrics output to compare raw relational processing speed versus using pre-aggregated view lookups.

```sql
-- Test 1: Query using base tables
\timing on
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
    JOIN products p ON s.product_id = p.product_id
    WHERE s.sale_date >= CURRENT_DATE - INTERVAL '12 months'
    GROUP BY p.category, s.region, DATE_TRUNC('month', s.sale_date)
) subquery
WHERE total_revenue > 10000
ORDER BY total_revenue DESC
LIMIT 20;

-- Test 2: Query using materialized view
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

### Step 2: 🛠️ Implement Performance Testing Function

**🚀 Implementation:** Write a helper function that evaluates and logs processing time across your test strings programmatically.

```sql
CREATE OR REPLACE FUNCTION test_query_performance(
    query_text TEXT,
    test_name TEXT
) RETURNS TABLE(test_name_out TEXT, execution_time_ms NUMERIC) AS $$
DECLARE
    start_time TIMESTAMPTZ;
    end_time TIMESTAMPTZ;
BEGIN
    start_time := clock_timestamp();
    EXECUTE query_text;
    end_time := clock_timestamp();
    
    test_name_out := test_name;
    execution_time_ms := EXTRACT(EPOCH FROM (end_time - start_time)) * 1000;
    RETURN NEXT;
END;
$$ LANGUAGE plpgsql;
```

### Step 3: 🏁 Run Comparative Tests

```sql
-- Test Materialized View Aggregation Performance
SELECT * FROM test_query_performance(
    'SELECT category, SUM(total_revenue) FROM mv_monthly_sales_summary GROUP BY category',
    'MV Aggregation'
);

-- Test Raw Table Aggregation Performance
SELECT * FROM test_query_performance(
    'SELECT p.category, SUM(s.quantity * p.price) FROM sales s JOIN products p ON s.product_id = p.product_id GROUP BY p.category',
    'Base Table Aggregation'
);
```

---

## 🔄 Task 4: Configure Refresh Strategies

### Step 1: 👆 Manual Refresh

> **ℹ️ Note:** Concurrent updates require an underlying `UNIQUE` index on the Materialized View target.

```sql
-- Complete rewrite block (Locks read access during refresh)
REFRESH MATERIALIZED VIEW mv_monthly_sales_summary;

-- Non-blocking refresh (Requires a unique index on the view)
-- Let's create a unique index first to support this
CREATE UNIQUE INDEX idx_mv_monthly_summary_unique ON mv_monthly_sales_summary(category, region, month);

REFRESH MATERIALIZED VIEW CONCURRENTLY mv_monthly_sales_summary;
```

### Step 2: ⚙️ Create Refresh Function

**🚀 Implementation:** Package view upkeep procedures into a single automated execution routine.

```sql
CREATE OR REPLACE FUNCTION refresh_all_sales_views() 
RETURNS TEXT AS $$
BEGIN
    -- Ensure unique indexes exist on dependencies to support CONCURRENTLY flags
    REFRESH MATERIALIZED VIEW CONCURRENTLY mv_monthly_sales_summary;
    REFRESH MATERIALIZED VIEW mv_recent_sales_summary; -- Regular refresh if no unique index exists
