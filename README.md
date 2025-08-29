# Hands ON materi Workshop Intermediate SQL
This Repository contain source code for Intermediate SQL using PostgreSQL in Online Data Engineering class.  

## Prerequisite
- PostgreSQL version 14 or above with include pgadmin. [Download](https://www.enterprisedb.com/downloads/postgres-postgresql-downloads)
- DBeaver. [Download](https://dbeaver.io/download/)
- VSCode (optional). [Download](https://code.visualstudio.com/download)
- Have knowledge Basic SQL
- Already have at least 1 Database Connection & 1 Database PostgreSQL in DBeaver. For detail please see [this step](https://github.com/data-engineers-id/dateng-nongki/tree/main/IntroSQL#1-create-new-database-connection)


## Setup Database Northwind
We will use Sample Database Northwind for this hands on. Please download [northwind.sql](https://github.com/pthom/northwind_psql/blob/master/northwind.sql) and run it in DBeaver


## 1) Hands On Conditional Logic Query
### 1.1 Simple Conditional Logic Query
Example : Categorize discount become "High", "Medium" or "Low" and calculate count of product for each category
```
SELECT
    CASE
        WHEN discount > 0.15 THEN 'High Discount'
        WHEN discount > 0.05 THEN 'Medium Discount'
        ELSE 'Low Discount'
    END AS discount_category,
    COUNT(product_id) AS total_products
FROM
    order_details
GROUP BY
    discount_category;
```

### 1.2 Advance Conditional Logic Query
Example : Customer Segmentation based on total purchase order
```
SELECT
    c.customer_id,
    c.contact_name,
    SUM(od.unit_price * od.quantity) AS total_order_value,
    CASE
        WHEN SUM(od.unit_price * od.quantity) > 10000 THEN 'Platinum'
        WHEN SUM(od.unit_price * od.quantity) BETWEEN 5001 AND 10000 THEN 'Gold'
        ELSE 'Silver'
    END AS customer_segment
FROM
    customers AS c
JOIN
    orders AS o ON c.customer_id = o.customer_id
JOIN
    order_details AS od ON o.order_id = od.order_id
GROUP BY
    c.customer_id, c.contact_name
ORDER BY
    total_order_value DESC;
```

## 2) Hands On SubQuery
### 2.1 Scalar SubQuery
Example: Find the product name with the highest price
```
SELECT p.product_name
FROM products p 
WHERE p.unit_price = (SELECT MAX(unit_price) FROM products);
```

### 2.2 Multi-row Subquery
Example : Find product that have been sold more than 50 units in 1 order
```
SELECT
    product_name
FROM
    products
WHERE
    product_id IN (SELECT product_id FROM order_details WHERE quantity > 50);
```

### 2.3 Correlated Subquery
Example : Find all product that have price more expensive that average price of product that have been sold by same supplier
```
SELECT
    product_name,
    unit_price
FROM
    products AS p1
WHERE
    unit_price > (
        SELECT AVG(unit_price)
        FROM products AS p2
        WHERE p2.supplier_id = p1.supplier_id
    );
```

### 2.4 Derived Table Subquery
Example : Find customer that have total order more than 10
```
SELECT
    c.contact_name,
    t.total_orders
FROM
    customers AS c
JOIN (
    SELECT
        customer_id,
        COUNT(order_id) AS total_orders
    FROM
        orders
    GROUP BY
        customer_id
    HAVING
        COUNT(order_id) > 10
) AS t ON c.customer_id = t.customer_id;
```

## 3) Hands On CTE Query
### 3.1 Simple CTE Query
Example : calculate total sales for each product category and display only category that have total sales more than average for all product
```
WITH CategorySales AS (
    SELECT
        c.category_name,
        SUM(od.unit_price * od.quantity) AS total_sales
    FROM
        categories AS c
    JOIN
        products AS p ON c.category_id = p.category_id
    JOIN
        order_details AS od ON p.product_id = od.product_id
    GROUP BY
        c.category_name
)
SELECT
    category_name,
    total_sales
FROM
    CategorySales
WHERE
    total_sales > (SELECT AVG(total_sales) FROM CategorySales);
```

### 3.2 Advance CTE Query
Example : Calculating total monthly sales and comparing them to sales in the same month of the previous year.
```
WITH MonthlySales AS (
    -- Langkah 1: Menghitung total penjualan bulanan
    SELECT
        EXTRACT(YEAR FROM o.order_date) AS sales_year,
        EXTRACT(MONTH FROM o.order_date) AS sales_month,
        SUM(od.unit_price * od.quantity * (1 - od.discount)) AS total_monthly_sales
    FROM
        orders AS o
    JOIN
        order_details AS od ON o.order_id = od.order_id
    GROUP BY
        sales_year, sales_month
),
YearlyComparison AS (
    -- Langkah 2: Menggunakan LAG() untuk mendapatkan penjualan tahun sebelumnya
    SELECT
        sales_year,
        sales_month,
        total_monthly_sales,
        LAG(total_monthly_sales, 12) OVER (PARTITION BY sales_month ORDER BY sales_year) AS previous_year_sales
    FROM
        MonthlySales
)
-- Langkah 3: Menghitung persentase pertumbuhan tahun-ke-tahun
SELECT
    sales_year,
    sales_month,
    total_monthly_sales,
    previous_year_sales,
    CASE
        WHEN previous_year_sales IS NULL THEN NULL -- Menghindari pembagian dengan nol atau data tidak tersedia
        ELSE (total_monthly_sales - previous_year_sales) / previous_year_sales * 100
    END AS yoy_growth_percentage
FROM
    YearlyComparison
ORDER BY
    sales_year, sales_month;
```

## 4) Window Function
### 4.1 Aggregate Window Function
Example : Calculating the Percentage of Product Sales from Total Category Sales
```
SELECT
    p.product_name,
    c.category_name,
    SUM(od.unit_price * od.quantity * (1 - od.discount)) AS product_sales,
    SUM(SUM(od.unit_price * od.quantity * (1 - od.discount))) OVER (PARTITION BY c.category_id) AS category_total_sales,
    (SUM(od.unit_price * od.quantity * (1 - od.discount)) / SUM(SUM(od.unit_price * od.quantity * (1 - od.discount))) OVER (PARTITION BY c.category_id)) * 100 AS percentage_of_category_sales
FROM
    products AS p
JOIN
    order_details AS od ON p.product_id = od.product_id
JOIN
    categories AS c ON p.category_id = c.category_id
GROUP BY
    p.product_name, c.category_name, c.category_id
ORDER BY
    c.category_name, percentage_of_category_sales DESC;
```

### 4.2 Ranking Window Function - RANK()
Example : Give rank to customer based on order count
```
SELECT
    customer_name,
    order_count,
    RANK() OVER (ORDER BY order_count DESC) AS customer_rank
FROM (
    SELECT
        c.contact_name customer_name,
        COUNT(order_id) AS order_count
    FROM
        orders o
    inner join customers c 
    on o.customer_id = c.customer_id
    GROUP BY
        customer_name
) AS customer_orders;
```

### 4.3 Ranking Window Function - ROW_NUMBER() 🔢
Examples : Assigning a row number to each order made by each customer, sorted by order date.
```
SELECT
    c.contact_name customer_name,
    o.ship_country, 
    o.order_date,
    ROW_NUMBER() OVER (PARTITION BY o.ship_country ORDER BY o.order_date) AS order_number
FROM
    orders o
inner join customers c 
on o.customer_id = c.customer_id;
```

### 4.4 Ranking Window Function - DENSE_RANK()
Examples : Ranking products based on total quantity sold
```
SELECT
    p.product_name,
    SUM(od.quantity) AS total_quantity_sold,
    sum(od.quantity * od.unit_price) as total_sales,
    DENSE_RANK() OVER (ORDER BY sum(od.quantity * od.unit_price) DESC) AS total_sales_rank
FROM
    products AS p
JOIN
    order_details AS od ON p.product_id = od.product_id
GROUP BY
    p.product_name
ORDER BY
    total_sales_rank;
```

### 4.5 Value Window Function - LAG() and LEAD()
Examples : Comparing monthly sales quantity with the previous (LAG) and next months (LEAD).
```
WITH MonthlySales AS (
    SELECT
        DATE_TRUNC('month', order_date) AS sales_month,
        SUM(od.quantity) AS monthly_quantity
    FROM
        orders AS o
    JOIN
        order_details AS od ON o.order_id = od.order_id
    GROUP BY
        sales_month
    ORDER BY
        sales_month
)
SELECT
    TO_CHAR(sales_month, 'MM-YYYY') AS sales_month,
    monthly_quantity,
    LAG(monthly_quantity, 1) OVER (ORDER BY sales_month) AS previous_month_quantity,
    LEAD(monthly_quantity, 1) OVER (ORDER BY sales_month) AS next_month_quantity
FROM
    MonthlySales;
```

### 4.6 Value Window Function - FIRST_VALUE() & LAST_VALUE()
Examples : Displaying the first and last order date for each customer next to every order they've made.
```
SELECT
    c.contact_name customer_name,
    o.ship_country,
    o.order_date,
    FIRST_VALUE(o.order_date) OVER (PARTITION BY o.ship_country ORDER BY o.order_date) AS first_order_date,
    LAST_VALUE(o.order_date) OVER (PARTITION BY o.ship_country ORDER BY o.order_date ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS last_order_date
FROM
    orders o
inner join customers c 
on o.customer_id = c.customer_id
ORDER BY
    customer_name, order_date;
```
