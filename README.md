# [SQL_Analysis_Project]
###  Performed end-to-end data analysis using SQL to clean, explore, and analyze a multi-table dataset. Created KPIs such as revenue, customer retention, product performance, and sales trends using joins, CTEs, and window functions. Delivered actionable insights to support data-driven decision-making and built a final summary report for stakeholders.
## Trend Analysis - (Growth Per Year)

### A trend analysis of year-over-year growth highlights how performance has changed annually and helps identify whether progress is accelerating, slowing, or remaining stable over time. By comparing the values from each year, we can observe patterns such as consistent growth, fluctuations, or periods of decline. This type of analysis makes it easier to understand long-term direction, evaluate the effectiveness of strategies, and forecast future outcomes. Overall, the year-by-year growth trend provides valuable insight into both the sustainability and momentum of performance over the analyzed period.
```sql
select YEAR(order_date) As Order_Year,
MONTH(order_date) As Order_Month,
sum(sales_amount) as Total_Sales,
COunt(Distinct customer_key) as Total_Customers,
sum(quantity) as Total_Quantity
from gold.fact_sales
where order_date is not null
Group by YEAR(order_date),MONTH(order_date)
order by YEAR(order_date),MONTH(order_date)
```
## Cumulative  Analysis- - (Progress Of bussiness - Running Growth Per Year)

### A cumulative analysis of the business’s running growth per year provides a comprehensive view of long-term progress by continuously adding each year’s results to the previous totals. Instead of evaluating individual yearly changes in isolation, this approach shows how the business has built momentum over time and how each year contributes to overall performance. By tracking cumulative growth, we can clearly identify upward or downward trajectories, understand the strength and sustainability of expansion, and assess how effectively the business is compounding its gains. This type of analysis offers a fuller picture of progress, helping in strategic planning, forecasting, and evaluating whether the business is moving steadily toward its long-term goals.

```sql
select order_date,
Total_Sales,
sum(Total_Sales) over (Order by order_date) as Running_Total_Sales,
Avg(Avg_Price) over (Order by order_date) as Moving_Average
from
(
select Datetrunc(YEAR,order_date) as Order_date,
sum(sales_amount) as Total_Sales,
avg(price) as Avg_Price
from gold.fact_sales 
where order_date is not null
group by Datetrunc(Year,order_date)
)t

```


----------Performance  Analysis----------(compare each year each product with previous year and avg of sales with all years.)

with yearly_product_sales as
(
select year(f.order_date) as order_year,p.product_name,sum(f.sales_amount) as Current_Sales
from gold.fact_sales f
left join gold.dim_products p 
on f.product_key=p.product_key
where f.order_date is not null
group by year(f.order_date),p.product_name
)
select order_year,product_name,Current_Sales,
avg(Current_Sales) over (partition by product_name) as avg_sales,
Current_Sales - avg(Current_Sales) over (partition by product_name) as Diff_Avg,
case when Current_Sales - avg(Current_Sales) over (partition by product_name) > 0 then 'Above Average'
     when Current_Sales - avg(Current_Sales) over (partition by product_name) < 0 then 'Below Average'
else 'Avg'
end avg_change,
lag(current_sales) over (partition by product_name order by order_year) prev_year_sales,
Current_Sales - lag(current_sales) over (partition by product_name order by order_year) diff_prev_year,
case when Current_Sales - lag(current_sales) over (partition by product_name order by order_year) > 0 then 'Incresed'
     when Current_Sales - lag(current_sales) over (partition by product_name order by order_year) < 0 then 'Decreased'
else 'No Change'
end Prev_change
from yearly_product_sales
order by product_name,order_year


----------Part to whole / Proportional  Analysis----------(each product sum / all product sum *100 -- How each category is contributing)

with cat_sales as
(
select p.category,sum(f.sales_amount) Total_Sales from gold.fact_sales f
left join gold.dim_products p
on f.product_key=p.product_key
group by p.category
)
select 
category,
Total_Sales,
sum(Total_Sales) over () as Overall_Sales,
Concat( round(( cast(Total_Sales as float) / sum(Total_Sales) over () ) * 100,2),'%') as percent_total
from cat_sales
order by Total_Sales desc

----------Part to whole / Proportional  Analysis----------(Best and least selling 2 products )

with cte_prod as
(
select category,product_name,sum(sales_amount) prods_Total_Sales from gold.fact_sales f
left join gold.dim_products p
on f.product_key=p.product_key
group by category,product_name
)
,
 cte_report as
(
select category,product_name,
prods_Total_Sales,
SUM(prods_Total_Sales) over (partition by category) as TotalSales,
Concat(Round((cast(prods_Total_Sales as float) / SUM(prods_Total_Sales) over (partition by category))*100,2), '%') as prod_per
from cte_prod
),
 cte_fr as
(
select category,product_name,prod_per,
Dense_Rank() over (partition by category order by prod_per ) as row_cat
from cte_report
)

select * from cte_fr
where row_cat < 3

----------Segment Analysis-------------------(1st)-----------------------------------------

with cte_product_segment as
(
select product_key,product_name,cost,
case when cost < 100 then 'Below 100'
     when cost between 100 and  500 then '100-500'
     when cost between 500 and  1000 then '500-1000'
else 'Above 1000'
end Cost_Range
from gold.dim_products
)

select Cost_Range,count(product_key) total_products
from cte_product_segment
group by Cost_Range 
order by total_products desc

----------Segment Analysis-------------------(2nd - on the basis on customer behaviour and their lifespan in company)-----------------------------------------
with cte_cusomer_lifespan as
(
select c.customer_key,
sum(s.sales_amount) as total_spending,
Min(order_date) as first_order,
Max(order_date) as last_order,
DATEDIFF(month,Min(order_date),Max(order_date)) as Life_Span
from gold.fact_sales s
left join gold.dim_customers c
on s.customer_key=c.customer_key
group by c.customer_key
)

select
Cust_segment,
count(customer_key) as Total_Customers
from (
select customer_key,
case when Life_Span >= 12 and total_spending >= 5000 then 'VIP'
	when Life_Span >= 12 and total_spending < 5000 then 'Regular'
	else 'NEW'
end Cust_segment 
from cte_cusomer_lifespan 
)t 
group by Cust_segment
order by Total_Customers

--------------------Final Analysis ------------------------------------------------------------
--------------------make data ready for further vizualization along with KPI------------------------------------------------------------
--------------------CUSTOMER KPI------------------------------------------------------------


--create view gold.V_Cust_Report as
with cte_base_query as
(
select 
f.order_number,
f.product_key,
f.order_date,
f.sales_amount,
f.quantity,
c.customer_key,
c.customer_number,
CONCAT(c.first_name, ' ' , c.last_name) customer_Name,
DATEDIFF(year,c.birthdate,GETDATE()) customer_Age
from gold.fact_sales f
left join gold.dim_customers c
on c.customer_key = f.customer_key
where order_date is not null
),
cte_cust_aggregation as
(
select 
customer_key,
customer_number,
customer_Name,
customer_Age,
Count(Distinct order_number) total_orders,
sum(sales_amount) total_sales,
sum(quantity) total_quantity,
Count(Distinct product_key) total_products,
max(order_date) last_order_date,
DATEDIFF(month,min(order_date),max(order_date)) lifespan
from cte_base_query
group by customer_key,
customer_number,
customer_Name,
customer_Age
)
select customer_key,
customer_number,
customer_Name,
customer_Age,
case when customer_Age < 20  then 'Under 20'
	 when customer_Age between 20 and 29  then '20-29'
	 when customer_Age between 30 and 39  then '30-39'
	 when customer_Age between 40 and 49  then '40-49'
	 else '50 and Above'
end as age_group,

case when lifespan >= 12 and total_sales >= 5000 then 'VIP'
	 when lifespan >= 12 and total_sales <= 5000 then 'REGULAR'
	 else 'NEW'
end as cust_segment,
total_orders,
total_sales,
total_quantity,
total_products,
last_order_date,
lifespan,
DATEDIFF(MONTH,last_order_date,getdate()) recency,  ----- KPI Last order to current lifespan------ 
case when total_sales = 0 then  0    ------------- KPI Avg Orders-----
else total_sales/total_orders 
end as avg_order_value,
case when lifespan = 0 then total_sales ------------KPI AVg Monthly spend ------------ 
else total_sales / lifespan
end as avg_monthly_spend
from cte_cust_aggregati



--select * from [gold].[V_Cust_Report] ------------Report in View ------------ 


----------Quick Analysis ------------ 
select cust_segment,
Count(customer_number) as total_customers,
sum(total_sales) as total_sales
from V_Cust_Report
group by cust_segment
order by total_sales desc


--------------------PRODUCT KPI------------------------------------------------------------

--create view gold.V_prod_Report as
with cte_base_prod as
(
select 
f.order_number,
f.order_date,
f.customer_key,
f.sales_amount,
f.quantity,
p.product_key,
p.product_name,
p.category,
p.subcategory,
p.cost
from gold.fact_sales f
left join gold.dim_products p
on f.product_key = p. product_key
where order_date is not null
),
cte_prod_aggregation as
(
select 
product_key,
product_name,
category,
subcategory,
cost,
Max(order_date) as last_order_date,
DATEDIFF(MONTH,min(order_date),max(order_date)) as Lifespan,
count(distinct order_number) as total_order,
count(distinct customer_key) as total_customers,
sum(sales_amount) as total_sales,
sum(quantity) as total_quantity,
Round(Avg(cast(sales_amount as float) / nullif(quantity,0)),1) as avg_selling_price
from cte_base_prod
group by product_key,
product_name,
category,
subcategory,
cost
)

select product_key,
product_name,
category,
subcategory,
cost,
last_order_date,
DATEDIFF(MONTH,last_order_date,GETDATE()) as recency_in_months,
case when total_sales > 50000 then 'High Performer'
	 when total_sales >= 10000 then 'Mid Performer'
	 else 'Low Performer'
end as product_segment,
Lifespan,
total_order,
total_sales,
total_quantity,
total_customers,
avg_selling_price,
case when total_order = 0  then  0
	 else total_sales / total_order
end as avg_order_revenue, ------------KPI--------------
case when Lifespan = 0  then  total_sales
	 else total_sales / Lifespan
end as avg_monthly_revenue------------KPI--------------
from cte_prod_aggregation

--------------------PRODUCT KPI------------------------------------------------------------

select product_segment,
Count(distinct product_key) as Total_products,
sum(total_sales) as Totla_sales
from [gold].[V_prod_Report]
group by product_segment
order by sum(total_sales)
