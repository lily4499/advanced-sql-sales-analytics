# advanced-sql-sales-analytics


---

## 🔥 Project Title: **Advanced Sales Analytics for a Global Retail Chain**

### 🧠 Objective:

Analyze and optimize global retail performance using advanced SQL techniques like **window functions**, **CTEs**, **subqueries**, **views**, **procedures**, and **performance tuning**.

---

### 📦 Dataset:

Use the **Global Superstore** dataset (more comprehensive than US-only). Download from:
[https://www.kaggle.com/datasets/vivek468/superstore-dataset-final](https://www.kaggle.com/datasets/vivek468/superstore-dataset-final)

```
mkdir advanced-sql-sales-analytics && cd advanced-sql-sales-analytics
python3 -m venv venv
source venv/bin/activate
pip install kaggle

mkdir data && cd data
kaggle datasets download ...
unzip ...
mv  ...  global_superstore.csv
```

---

### 🗂️ Project Structure:

```
advanced-sql-sales-analytics/
│
├── data/
│   └── global_superstore.csv
│
├── sql/
│   ├── create_tables.sql
│   ├── load_data.sql
│   ├── transformations.sql
│   ├── analysis_queries.sql
│   └── procedures_and_views.sql
│
└── reports/
    └── sales_insights_report.pdf
```

---

## 🔧 Step-by-Step Instructions:

---

### 1️⃣ Data Cleaning with Python (optional but recommended)


```bash
 pip install pandas chardet
```

# Inspect the actual column names
```
import pandas as pd

df = pd.read_csv('data/global_superstore.csv', encoding='latin1')
print(df.columns.tolist())

```

# clean_csv.py

```
import pandas as pd
import chardet

# Step 1: Detect encoding
with open('data/global_superstore.csv', 'rb') as f:
    raw_data = f.read()
    encoding = chardet.detect(raw_data)['encoding']
print(f"✅ Detected Encoding: {encoding}")

# Step 2: Load the CSV using the detected encoding
df = pd.read_csv('data/global_superstore.csv', encoding=encoding)

# Step 3: Clean column names (strip whitespace)
df.columns = df.columns.str.strip()

# Step 4: Clean non-ASCII characters (e.g., �)
df = df.astype(str).map(lambda x: x.encode('ascii', 'ignore').decode('ascii')).apply(pd.Series)

# Step 5: Convert 'Order Date' and 'Ship Date' to MySQL-compatible format
df['Order Date'] = pd.to_datetime(df['Order Date'], errors='coerce')
df['Ship Date'] = pd.to_datetime(df['Ship Date'], errors='coerce')
df['Order Date'] = df['Order Date'].dt.strftime('%Y-%m-%d')
df['Ship Date'] = df['Ship Date'].dt.strftime('%Y-%m-%d')

# Step 6: Clean numeric fields
df['Sales'] = df['Sales'].replace(r'[\$,]', '', regex=True).replace(',', '', regex=True).astype(float).round(2)
df['Profit'] = df['Profit'].replace(r'[\$,]', '', regex=True).replace(',', '', regex=True).astype(float).round(2)
df['Discount'] = df['Discount'].replace(r'[\%,]', '', regex=True).astype(float).round(2)

# Step 7: Save the cleaned data
df.to_csv('data/global_superstore_cleaned.csv', index=False)
print("✅ Final cleaned CSV saved at: data/global_superstore_cleaned.csv")

```
python3 clean_csv.py

---

### 2️⃣ MySQL Setup

**Create Database**

```sql
mysql -u root -p
GRANT ALL PRIVILEGES ON *.* TO 'lili'@'localhost' WITH GRANT OPTION;
FLUSH PRIVILEGES;
exit

mysql -u lili -p
CREATE DATABASE superstore_analytics;
USE superstore_analytics;
```

---

### 3️⃣ Create Tables (`create_tables.sql`)

Example:

```sql
CREATE TABLE orders (
    row_id INT PRIMARY KEY,
    order_id VARCHAR(20),
    order_date DATE,
    ship_date DATE,
    ship_mode VARCHAR(50),
    customer_id VARCHAR(20),
    customer_name VARCHAR(100),
    segment VARCHAR(50),
    country VARCHAR(50),
    city VARCHAR(50),
    state VARCHAR(50),
    postal_code VARCHAR(20),
    region VARCHAR(50),
    product_id VARCHAR(20),
    category VARCHAR(50),
    sub_category VARCHAR(50),
    product_name TEXT,
    sales DECIMAL(10,2),
    quantity INT,
    discount DECIMAL(5,2),
    profit DECIMAL(10,2)
);
```

---

### 4️⃣ Load Data (`load_data.sql`)
```
sudo cp data/global_superstore_cleaned.csv /var/lib/mysql-files/
```

```sql
LOAD DATA INFILE '/var/lib/mysql-files/global_superstore_cleaned.csv'
INTO TABLE orders
FIELDS TERMINATED BY ',' ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS;
```

### Troubleshooting

```
SHOW WARNINGS LIMIT 10;
DESCRIBE orders;
python3 clean_csv.py
sudo cp data/global_superstore_cleaned.csv /var/lib/mysql-files/
TRUNCATE TABLE orders;

LOAD DATA INFILE '/var/lib/mysql-files/global_superstore_cleaned.csv'
INTO TABLE orders
FIELDS TERMINATED BY ',' ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS;




```


✅ *Tip*: Use Workbench's “Table Data Import Wizard” if you face permission issues.

---

### 5️⃣ Transformations (`transformations.sql`)

```sql
-- Add month and year columns
ALTER TABLE orders ADD COLUMN order_month VARCHAR(20), ADD COLUMN order_year INT;

UPDATE orders
SET order_month = MONTHNAME(order_date),
    order_year = YEAR(order_date);
```

---

### 6️⃣ Analysis Queries (`analysis_queries.sql`)

#### A. Top 5 most profitable products (Window function):

```sql
SELECT product_name, SUM(profit) AS total_profit,
       RANK() OVER (ORDER BY SUM(profit) DESC) AS rank
FROM orders
GROUP BY product_name
LIMIT 5;
```

#### B. Monthly trend of sales and profit (CTE):

```sql
WITH monthly_sales AS (
    SELECT order_year, order_month,
           SUM(sales) AS total_sales, SUM(profit) AS total_profit
    FROM orders
    GROUP BY order_year, order_month
)
SELECT * FROM monthly_sales ORDER BY order_year, FIELD(order_month, 
    'January','February','March','April','May','June','July',
    'August','September','October','November','December');
```

#### C. Customer segmentation:

```sql
SELECT customer_id, customer_name,
       SUM(sales) AS total_sales,
       COUNT(DISTINCT order_id) AS num_orders,
       CASE
           WHEN SUM(sales) > 10000 THEN 'High-Value'
           WHEN SUM(sales) > 5000 THEN 'Mid-Value'
           ELSE 'Low-Value'
       END AS segment
FROM orders
GROUP BY customer_id, customer_name;
```

#### D. Detect loss-making products (subqueries):

```sql
SELECT product_name, category, SUM(profit) AS total_profit
FROM orders
GROUP BY product_name, category
HAVING total_profit < 0;
```

---

### 7️⃣ Views and Stored Procedures (`procedures_and_views.sql`)

#### Create View:

```sql
CREATE VIEW sales_summary AS
SELECT region, category, SUM(sales) AS sales, SUM(profit) AS profit
FROM orders
GROUP BY region, category;
```

#### Stored Procedure:

```sql
DELIMITER //

CREATE PROCEDURE get_customer_orders(IN cid VARCHAR(20))
BEGIN
    SELECT order_id, order_date, product_name, sales, profit
    FROM orders
    WHERE customer_id = cid;
END //

DELIMITER ;
```

---

## 📊 Deliverable: Report in PDF

Create a dashboard or export insights as PDF with:

* KPIs (total sales, profit, avg discount)
* Visuals (monthly trend, category performance)
* Segment analysis (high-value customers)
* Recommendations

---

## ✅ SQL Skills Demonstrated:

| Skill                          | Used? |
| ------------------------------ | ----- |
| Joins                          | ✅     |
| Window functions               | ✅     |
| Subqueries & Nested SELECTs    | ✅     |
| Views                          | ✅     |
| Stored Procedures              | ✅     |
| Aggregations & Grouping        | ✅     |
| CTEs                           | ✅     |
| Case statements                | ✅     |
| Data cleaning & transformation | ✅     |

---

## 🧩 Bonus Ideas:

* Add `shipping_days` column to calculate `DATEDIFF(ship_date, order_date)`
* Export results using `SELECT ... INTO OUTFILE`
* Connect MySQL with Power BI or Tableau for live dashboards

---

