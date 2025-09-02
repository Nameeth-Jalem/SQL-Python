#  Retail orders Analysis using SQL and Python 

### End-to-End Data Analytics Project (Kaggle API + Python + SQL)
Project Overview

This project demonstrates an end-to-end data analytics pipeline  starting with data extraction from Kaggle, cleaning and preprocessing with Python, and performing analysis in SQL Server to derive business insights.

Tech Stack

Kaggle API - Dataset extraction

Python (Pandas, NumPy, SQLAlchemy, PyODBC)  Data preprocessing, cleaning, and SQL connection

SQL Server -  Data modeling and analysis queries



Situation:
Organizations rely on structured insights to understand sales, products, and customer behavior. Raw datasets from external sources are often unclean and unsuitable for direct analysis.

Task:
My role was to design a process that extracts raw data, cleans and transforms it for consistency, loads it into SQL, and applies queries to answer business questions.

Action:

Extracted dataset using Kaggle API.

Cleaned and preprocessed data in Python:

Renamed columns to follow best practices.

Derived new columns (discount, sale_price, profit).

Converted order_date to datetime format.

Dropped unnecessary columns.

Exported cleaned data to CSV.

Established SQL Server connection using SQLAlchemy + PyODBC.

Created a relational table (df_orders) and inserted cleaned data.

Ran multiple SQL queries to answer business questions.

Result:
Generated insights on top-selling products, regional performance, month-over-month trends, and category growth — enabling better decision-making for business stakeholders.

Python Preprocessing Snippet
import pandas as pd

## Data Preprocessing
```
df = pd.read_csv(r"D:\CodeBasics Datasets\orders.csv",
                 encoding='ISO-8859-1',
                 na_values=['Not Available', 'unknown'])

# Rename columns (snake_case)
df.columns = df.columns.str.lower().str.replace(' ', '_')

# Derive new columns
df['discount'] = df['list_price'] * df['discount_percent'] * 0.01
df['sale_price'] = df['list_price'] - df['discount']
df['profit'] = df['sale_price'] - df['cost_price']

# Convert order_date to datetime
df['order_date'] = pd.to_datetime(df['order_date'], format="%Y-%m-%d")

# Drop unnecessary columns
df.drop(columns=['list_price', 'discount_percent', 'cost_price'], inplace=True)

# Save cleaned dataset
cleaned_df = df.copy()
cleaned_df.to_csv("cleaned_df.csv", index=False)

SQL Server Connection Snippet
# Install dependencies
pip install sqlalchemy
pip install pyodbc

import urllib
import sqlalchemy as sal

params = urllib.parse.quote_plus(
    "DRIVER={ODBC Driver 18 for SQL Server};"
    "SERVER=NAMEETH\\SQLEXPRESS;"
    "DATABASE=master;"
    "Trusted_Connection=yes;"
    "Encrypt=no;"
)

# Create engine
engine = sal.create_engine(f"mssql+pyodbc:///?odbc_connect={params}")

# Test connection
with engine.connect() as conn:
    print("Connection successful")

# Load data into SQL table
df.to_sql('df_orders', con=engine, index=False, if_exists='append')
```
## SQL Script
```
SQL Table Definition
CREATE TABLE df_orders (
    order_id INT PRIMARY KEY,
    order_date DATE,
    ship_mode VARCHAR(20),
    segment VARCHAR(20),
    country VARCHAR(20),
    city VARCHAR(20),
    state VARCHAR(20),
    postal_code VARCHAR(20),
    region VARCHAR(20),
    category VARCHAR(20),
    sub_category VARCHAR(20),
    product_id VARCHAR(50),
    quantity INT,
    discount DECIMAL(7,2),
    sale_price DECIMAL(7,2),
    profit DECIMAL(7,2)
);
```
SQL Analysis – Questions and Answers (13)
Q1. Find top 10 highest generating products ?
```
SELECT TOP 10 product_id, SUM(sale_price) AS sales
FROM df_orders
GROUP BY product_id
ORDER BY sales DESC;
```

Q2. Find region-wise highest generating products?
```
WITH cte AS (
    SELECT region, product_id, SUM(sale_price) AS sales
    FROM df_orders
    GROUP BY product_id, region
)
SELECT *
FROM (
    SELECT *, ROW_NUMBER() OVER(PARTITION BY region ORDER BY sales DESC) AS rnk
    FROM cte
) A
WHERE rnk <= 5;
```

Q3. Month-over-month sales comparison (2022 vs 2023) ?
```
WITH cte AS (
    SELECT YEAR(order_date) AS oy,
           MONTH(order_date) AS om,
           SUM(sale_price) AS sales
    FROM df_orders
    GROUP BY YEAR(order_date), MONTH(order_date)
)
SELECT om,
       SUM(CASE WHEN oy = 2022 THEN sales ELSE 0 END) AS sales_2022,
       SUM(CASE WHEN oy = 2023 THEN sales ELSE 0 END) AS sales_2023
FROM cte
GROUP BY om
ORDER BY om;
```

Q4. Category-wise highest sales month
```
WITH cte AS (
    SELECT category, FORMAT(order_date, 'yyyyMM') AS o_y_m, SUM(sale_price) AS sales
    FROM df_orders
    GROUP BY category, FORMAT(order_date, 'yyyyMM')
)
SELECT *
FROM (
    SELECT *, ROW_NUMBER() OVER(PARTITION BY category ORDER BY sales DESC) AS rnk
    FROM cte
) a
WHERE rnk = 1;
```

Q5. Sub-category with highest growth (2022 → 2023) ?
```
WITH cte AS (
    SELECT sub_category, YEAR(order_date) AS oy, SUM(sale_price) AS sales
    FROM df_orders
    GROUP BY sub_category, YEAR(order_date)
),
cte2 AS (
    SELECT sub_category,
           SUM(CASE WHEN oy = 2022 THEN sales ELSE 0 END) AS sales_2022,
           SUM(CASE WHEN oy = 2023 THEN sales ELSE 0 END) AS sales_2023
    FROM cte
    GROUP BY sub_category
)
SELECT TOP 2 *, (sales_2023 - sales_2022) * 100 / sales_2022 AS growth_percentage
FROM cte2
ORDER BY growth_percentage DESC;
```

Q6. Distinct regions present in dataset?

```
SELECT DISTINCT region
FROM df_orders;
```

Q7. Distinct years present in dataset?
```
SELECT DISTINCT YEAR(order_date)
FROM df_orders;
```

Q8. Total sales and profit by region ?
```
SELECT region, SUM(sale_price) AS total_sales, SUM(profit) AS total_profit
FROM df_orders
GROUP BY region
ORDER BY total_sales DESC;
```

Q9. Top 3 categories by average sales per transaction ?
```

SELECT category, AVG(sale_price) AS avg_sales
FROM df_orders
GROUP BY category
ORDER BY avg_sales DESC
OFFSET 0 ROWS FETCH NEXT 3 ROWS ONLY;
```

Q10. Top 5 customers by total purchase value ?
```

SELECT customer_id, SUM(sale_price) AS total_purchase
FROM df_orders
GROUP BY customer_id
ORDER BY total_purchase DESC
OFFSET 0 ROWS FETCH NEXT 5 ROWS ONLY;
```

Q11. Category with maximum number of unique products?
```
SELECT category, COUNT(DISTINCT product_id) AS unique_products
FROM df_orders
GROUP BY category
ORDER BY unique_products DESC;
```

Q12. Average order value (AOV)?
```

SELECT AVG(order_total) AS avg_order_value
FROM (
    SELECT order_id, SUM(sale_price) AS order_total
    FROM df_orders
    GROUP BY order_id
) AS order_summary;
```

Q13. Product with highest total sales revenue?
```
SELECT product_id, SUM(sale_price) AS total_revenue
FROM df_orders
GROUP BY product_id
ORDER BY total_revenue DESC;
```

Key Insights

The top 10 products contribute a significant share of total revenue.

Certain regions consistently outperform others, with some products being region-specific leaders.

Month-over-month analysis reveals seasonal trends and growth patterns from 2022 to 2023.

Specific sub-categories show exceptional year-over-year growth, indicating areas for expansion.

The top customers contribute disproportionately high revenue, highlighting the importance of retention strategies.
