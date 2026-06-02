# 📊 OLAP Analytics Implementation with ClickHouse

<div align="center">

# ⚡ ClickHouse OLAP Analytics 

### 🧠 SQL Analytics • 🚀 Columnar Storage • 📊 OLAP Queries • ⚙️ Performance Optimization

![ClickHouse](https://img.shields.io/badge/ClickHouse-OLAP-FD7F20?style=for-the-badge&logo=clickhouse&logoColor=white)
![SQL](https://img.shields.io/badge/SQL-Analytics-0064A5?style=for-the-badge&logo=databricks&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-Ubuntu-FCC624?style=for-the-badge&logo=linux&logoColor=black)
![Database](https://img.shields.io/badge/Database-OLAP-blue?style=for-the-badge)

</div>

---

# 📖 Overview

This lab provides a **hands-on implementation of OLAP (Online Analytical Processing)** using **ClickHouse**, a high-performance columnar database designed for real-time analytics at scale.

You will design OLAP-optimized schemas, load sample sales data, and execute advanced analytical queries including aggregations, joins, and window functions.

---

# 🎯 Learning Objectives

By completing this lab, you will:

✅ Install and configure ClickHouse on Linux  
✅ Design OLAP-optimized fact and dimension tables  
✅ Execute high-performance analytical SQL queries  
✅ Use window functions for advanced analytics  
✅ Create materialized views for performance optimization  
✅ Understand columnar storage benefits for OLAP workloads  

---

# 📋 Prerequisites

- 🐧 Linux command-line basics  
- 🧠 Understanding of SQL (SELECT, JOIN, GROUP BY)  
- 📊 Basic database concepts  
- ⚡ Familiarity with analytical queries  
- 🧾 Understanding of data modeling concepts  

---

# 🛠️ Technology Stack

| Technology | Purpose |
|------------|---------|
| ClickHouse | OLAP Columnar Database |
| SQL | Query Language |
| Linux | System Environment |
| MergeTree Engine | High-performance storage engine |
| Materialized Views | Query acceleration |

---

# ⚙️ Environment Setup

---

## 🔹 Step 1: Install Dependencies

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates dirmngr
```

---

## 🔹 Step 2: Add ClickHouse Repository

```bash
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 8919F6BD2B48D754

echo "deb https://packages.clickhouse.com/deb stable main" | sudo tee /etc/apt/sources.list.d/clickhouse.list
```

---

## 🔹 Step 3: Install ClickHouse

```bash
sudo apt-get update
sudo apt-get install -y clickhouse-server clickhouse-client
```

---

## 🔹 Step 4: Start ClickHouse

```bash
sudo service clickhouse-server start
sudo service clickhouse-server status
```

---

## 🔹 Step 5: Connect to ClickHouse

```bash
clickhouse-client
```

You should see:

```text
localhost :)
```

---

# 🧪 Task 1: Create OLAP Database

---

## 🔹 Step 1: Create Database

```sql
CREATE DATABASE IF NOT EXISTS sales_analytics;
USE sales_analytics;
```

---

# 🏗️ Task 2: OLAP Table Design

---

## 📊 Fact Table (Sales Data)

```sql
CREATE TABLE sales_fact
(
    sale_id UInt64,
    sale_date Date,
    product_id UInt32,
    customer_id UInt32,
    region String,
    quantity UInt16,
    unit_price Decimal(10, 2),

    -- Calculated fields (OLAP optimization idea)
    discount_percent UInt8,

    payment_method String
)
ENGINE = MergeTree()
ORDER BY (sale_date, region, product_id);
```

---

## 📦 Product Dimension Table

```sql
CREATE TABLE products
(
    product_id UInt32,
    product_name String,
    category String,
    subcategory String,
    brand String
)
ENGINE = MergeTree()
ORDER BY product_id;
```

---

## 👤 Customer Dimension Table

```sql
CREATE TABLE customers
(
    customer_id UInt32,
    customer_name String,
    customer_segment String,
    registration_date Date,
    country String
)
ENGINE = MergeTree()
ORDER BY customer_id;
```

---

# 📥 Sample Data Insertion

## 📦 Insert Products

```sql
INSERT INTO products VALUES
(1, 'Laptop Pro 15', 'Electronics', 'Computers', 'TechBrand'),
(2, 'Wireless Mouse', 'Electronics', 'Accessories', 'TechBrand'),
(3, 'Office Chair', 'Furniture', 'Seating', 'ComfortCo'),
(4, 'Desk Lamp', 'Furniture', 'Lighting', 'BrightLight'),
(5, 'USB-C Cable', 'Electronics', 'Accessories', 'TechBrand');
```

---

## 👤 Insert Customers

```sql
INSERT INTO customers VALUES
(1, 'John Doe', 'Premium', '2023-01-15', 'USA'),
(2, 'Alice Smith', 'Standard', '2023-03-10', 'UK'),
(3, 'Robert Brown', 'Basic', '2023-06-05', 'Canada'),
(4, 'Emma Wilson', 'Premium', '2023-02-20', 'Germany'),
(5, 'David Lee', 'Standard', '2023-04-18', 'France');
```

---

## 💰 Insert Sales Data

```sql
INSERT INTO sales_fact VALUES
(1, '2024-01-15', 1, 1, 'North', 2, 1200.00, 5, 'Credit Card'),
(2, '2024-01-16', 2, 2, 'South', 5, 25.00, 0, 'PayPal'),
(3, '2024-01-17', 3, 3, 'East', 1, 300.00, 10, 'Cash'),
(4, '2024-01-18', 4, 4, 'West', 3, 80.00, 5, 'Credit Card'),
(5, '2024-01-19', 5, 5, 'North', 10, 15.00, 0, 'Debit Card');
```

---

# 📊 Task 3: OLAP Analytical Queries

---

## 📈 Query 1: Sales by Region

```sql
SELECT
    region,
    sum(quantity) AS total_quantity,
    sum(quantity * unit_price) AS total_revenue,
    avg(quantity * unit_price) AS avg_order_value
FROM sales_fact
GROUP BY region
ORDER BY total_revenue DESC;
```

---

## 📆 Query 2: Monthly Sales Trend

```sql
SELECT
    toStartOfMonth(sale_date) AS month,
    count() AS total_transactions,
    sum(quantity * unit_price) AS total_sales,
    avg(quantity * unit_price) AS avg_transaction_value
FROM sales_fact
GROUP BY month
ORDER BY month;
```

---

## 🔗 Query 3: Product Category + Region Analysis

```sql
SELECT
    p.category,
    s.region,
    sum(s.quantity * s.unit_price) AS revenue,
    sum(s.quantity) AS units_sold,
    avg(s.discount_percent) AS avg_discount
FROM sales_fact s
JOIN products p ON s.product_id = p.product_id
GROUP BY p.category, s.region
HAVING revenue > 1000
ORDER BY revenue DESC;
```

---

## 📊 Query 4: Window Function (Ranking + Running Total)

```sql
SELECT
    sale_date,
    region,
    sum(quantity * unit_price) AS daily_sales,

    sum(sum(quantity * unit_price)) OVER
    (PARTITION BY region ORDER BY sale_date) AS running_total,

    rank() OVER
    (PARTITION BY region ORDER BY sum(quantity * unit_price) DESC) AS sales_rank
FROM sales_fact
GROUP BY sale_date, region
ORDER BY region, sale_date;
```

---

# ⚡ Task 4: Materialized View

---

## 🚀 Create Materialized View

```sql
CREATE MATERIALIZED VIEW daily_sales_summary
ENGINE = SummingMergeTree()
ORDER BY (sale_date, region)
AS
SELECT
    sale_date,
    region,
    sum(quantity * unit_price) AS total_sales,
    count() AS transaction_count
FROM sales_fact
GROUP BY sale_date, region;
```

---

## 📊 Query Materialized View

```sql
SELECT *
FROM daily_sales_summary
WHERE sale_date >= today() - 30;
```

---

# 🔍 Verification

## Check Tables

```sql
SHOW TABLES FROM sales_analytics;
```

---

## Check Data

```sql
SELECT count() FROM sales_fact;
SELECT count() FROM products;
SELECT count() FROM customers;
```

---

## Performance Check

```sql
SELECT
    region,
    sum(quantity * unit_price) AS revenue
FROM sales_fact
GROUP BY region;
```

---

# 🧰 Troubleshooting

---

## ❌ Service Not Starting

```bash
sudo tail -f /var/log/clickhouse-server/clickhouse-server.log
```

---

## ❌ Connection Issue

```bash
sudo service clickhouse-server restart
```

---

## ❌ Query Errors

- Check column names (case-sensitive)
- Validate data types in INSERT statements
- Ensure JOIN keys match

---

## ⚡ Performance Issues

- Optimize ORDER BY clause
- Use PREWHERE for filtering
- Check query plan using EXPLAIN

---

# 📈 Expected Outcomes

After completing this lab, you will have:

✅ ClickHouse installed and running  
✅ OLAP optimized schema (fact + dimension tables)  
✅ Sample sales dataset loaded  
✅ Advanced analytical SQL queries executed  
✅ Materialized views for performance optimization  

---

# 🧠 Key Takeaways

### 📦 Columnar Storage
- Optimized for analytics
- Reads only required columns

### ⚡ MergeTree Engine
- High-performance storage
- Supports large-scale OLAP workloads

### 📊 OLAP Queries
- Aggregations
- Window functions
- Multi-dimensional analysis

### 🚀 Materialized Views
- Precomputed results
- Faster query performance

---

# 🚀 Next Steps

- Explore Distributed ClickHouse Clusters
- Implement Real-Time Streaming Inserts
- Try Advanced Aggregation Functions
- Work with Time-Series Analytics
- Integrate ClickHouse with Kafka
- Explore Performance Profiling Tools

---

# 🏁 Conclusion

You have successfully implemented a **full OLAP analytics system using ClickHouse**.

This lab demonstrated how modern analytical databases achieve **millisecond-level query performance** using:

- Columnar storage
- Smart indexing (ORDER BY)
- Materialized views
- Efficient aggregation engines

ClickHouse is widely used in production systems at companies like **Cloudflare, Uber, and Yandex** for real-time analytics at scale.

---

<div align="center">

### 🎉 OLAP ClickHouse Lab Completed Successfully!

⚡📊🚀 **Fast Analytics • Scalable Design • Real-Time Insights**

</div>
