# Insights-for-Sales-team-SQL
* Provide the list of markets in which customer "Atliq Exclusive" operates its business in the APAC region */

SELECT market FROM dim_customer
WHERE customer = "Atliq Exclusive" AND region = "APAC";

/*What is the percentage of unique product increase in 2021 vs. 2020?*/

SELECT unique_products_2020, unique_products_2021, ( unique_products_2021 / unique_products_2020 - 1) * 100.0 as percentage_chg from 
(Select count( DISTINCT product_code) as unique_products_2021, fiscal_year,
LAG(COUNT( DISTINCT product_code), 1) OVER(ORDER BY fiscal_year) as unique_products_2020
FROM fact_sales_monthly
GROUP BY fiscal_year) as sub
WHERE fiscal_year = 2021;

/*Provide a report with all the unique product counts for each segment and sort them in descending order of product counts*/

SELECT segment, COUNT(DISTINCT product) as num_unique
FROM dim_product
GROUP BY segment
ORDER BY num_unique DESC;

/*Get the products that have the highest and lowest manufacturing costs.*/

SELECT * 
-- Highest cost
FROM (SELECT product_code, product, MAX(manufacturing_cost) as cost
FROM fact_manufacturing_cost as cost
INNER JOIN dim_product as product USING(product_code)
GROUP BY product_code, product
ORDER BY cost DESC
LIMIT 1)
 -- Combine with lowest cost
 UNION
(SELECT product_code, product, MIN(manufacturing_cost) as cost
FROM fact_manufacturing_cost as cost
INNER JOIN dim_product as product USING(product_code)
GROUP BY product_code, product
ORDER BY cost
LIMIT 1);

/*Which contains the top 5 customers who received an average high pre_invoice_discount_pct for the fiscal year 2021 and in the Indian market.*/

SELECT customer_code, customer, pre_invoice_discount_pct
FROM fact_pre_invoice_deductions as invoice 
INNER JOIN dim_customer as customer USING( customer_code)
WHERE market = "India" AND fiscal_year = 2021
ORDER BY pre_invoice_discount_pct DESC
LIMIT 5;

/*Get the complete report of the Gross sales amount for the customer “Atliq Exclusive” for each month.*/

/*This analysis helps to get an idea of low and high-performing months and take strategic decisions.*/

SELECT EXTRACT(month from date) as month, EXTRACT(year from date) as year,
-- calculates the gross sale amount
SUM(ROUND(gross_price * sold_quantity)) as gross_amount
FROM dim_customer as customer
INNER JOIN fact_sales_monthly as sales USING(customer_code)
INNER JOIN fact_gross_price as price ON sales.product_code = price.product_code
where customer = 'Atliq Exclusive'
GROUP BY month, year
ORDER BY year, month;

/*In which quarter of 2020, got the maximum total_sold_quantity?*/

SELECT quarter, SUM(sold_quantity) as total_sold_quantity
-- creates the quarter column
FROM (SELECT *, EXTRACT(quarter from date) as quarter
FROM fact_sales_monthly
WHERE EXTRACT( year from date) = 2020) as sub
GROUP BY quarter 
ORDER BY total_sold_quantity DESC;

/*Which channel helped to bring more gross sales in the fiscal year 2021 and the percentage of contribution?*/

SELECT channel, gross_amount,gross_amount / 
-- Complete the formula by calculating the sum of gross sales
(SELECT SUM(gross_amount) FROM (SELECT channel, SUM(ROUND(gross_price * sold_quantity)) as gross_amount
FROM dim_customer as customer
INNER JOIN fact_sales_monthly as sales USING(customer_code)
INNER JOIN fact_gross_price as price ON sales.product_code = price.product_code
GROUP BY channel) as sub) * 100.0 as percentage 
-- Subquery to calculate gross sales 
FROM (SELECT channel, SUM(ROUND(gross_price * sold_quantity)) as gross_amount
FROM dim_customer as customer
INNER JOIN fact_sales_monthly as sales USING(customer_code)
INNER JOIN fact_gross_price as price ON sales.product_code = price.product_code
GROUP BY channel) as sub
GROUP BY channel;

/*Get the Top 3 products in each division that have a high total_sold_quantity in the fiscal_year 2021?*/

 -- creates ranking column
WITH cte as(SELECT *, RANK() OVER(PARTITION BY division ORDER BY total_sold_quantity DESC) as rank_order 
-- calculates total sold quantity in each division 
FROM (SELECT division, product_code, product, SUM(sold_quantity) as total_sold_quantity
FROM fact_sales_monthly as sales
INNER JOIN dim_product as product USING(product_code)
GROUP BY division, product_code, product) as sub)

SELECT * FROM cte
WHERE rank_order <= 3
