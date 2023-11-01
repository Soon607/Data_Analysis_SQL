# Trending the Data
* Looking for trends in the data when dealing with series data
* Trend: The direction in which the data is moving(moving up, increasing over time, moving down, decreasing over time)

## The Retail Sales Data Set
![The Retail Sales Data Set](https://github.com/Soon607/Data-Analysis-SQL-/assets/98214539/9a61ff03-145e-4ef0-8912-c268ffefd524)
## Simple Trends
* Creating a trend: A step in profiling and understanding data or final output
* The result set is a series of dates or timestamps and a numerical value.
* x-axis: **the dates** or **timestamps**
* y-axis: **numerical value** 
```sql
select sales_month,sales
from retail_sales
where kind_of_business='Retail and food services sales, total';
```
  * This data has some patterns, but it also has some noise
  * **Aggregating** at the yearly level can help gain a better understanding

```sql
---PostgreSql
select date_part('year',sales_month) as sales_year
,sum(sales) as sales
from retail_sales
where kind_of_business='Retail and food services sales, total'
group by 1
order by sales_year;
```
  * using `date_part` to return the year from sales_month
  * If we visualise the above output, we will have a **smoother time series**

      
*Graphing times series data at different levels of aggregation(weekly, monthly, yearly) is a good way to understand trends.*

## Comparing Components
* Dealing with the data that contains **multiple slices** or **components** of a total across the same time range.
1. Comparing the yearly sales trend for a few categories(book stores, sporting goods stores, hobby stores)
```sql
---PostgreSql
select date_part('year',sales_month) as sales_year,
kind_of_business,
sum(sales) as sales
from retail_sales
where kind_of_business in ('Book stores','Sporting goods stores', 'Hobby, toy, and games stores')
group by 1,2
order by sales_year
```
2. Looking at sales at **women's clothing stores** and **men's clothing stores**
  1) **Trending** the data for each type of store by month
     ```sql
     ---PostgreSql
     select
     sales_month,
     kind_of_business,
     sales
     from retail_sales
     where kind_of_business in('Men''s clothing stores','Women''s clothing stores')
     order by sales_month;
     ```
  2) **Aggregating** at the yearly level to eliminate noise in the pattern
     ```sql
     ---PostgreSql
     select
      date_part('year',sales_month) as sales_year,
      kind_of_business,
      sum(sales) as sales
      from retail_sales
      where kind_of_business in('Men''s clothing stores','Women''s clothing stores')
      group by 1,2
      order by sales_year;
     ```
  3) **Calculating** the gap between the two categories(ratio, percent difference between them)
     * First step: Arranging the data so that there is a single row for each year, with a column for each category(**Pivoting** the data)
       ```sql
       ---PostgreSql
       select
        date_part('year',sales_month) as sales_year,
        sum(case when kind_of_business='Women''s clothing stores' then sales end) as womens_sales,
        sum(case when kind_of_business='Men''s clothing stores' then sales end) as mens_sales
        from retail_sales
        where kind_of_business in ('Women''s clothing stores','Men''s clothing stores')
        group by 1
        order by sales_year;
       ```
     * Second step: Building block calculation (Finding the difference, ratio, percent difference between time series)
       1. **Difference**
          * using `subquery`
          ```sql
          ---PostgreSql
          select
            sales_year,
            womens_sales-mens_sales as womens_minus_mens,
            mens_sales-womens_sales as mens_minus_womens
            from
            (select
            date_part('year',sales_month) as sales_year,
            sum(case when kind_of_business='Women''s clothing stores' then sales end) as womens_sales,
            sum(case when kind_of_business='Men''s clothing stores' then sales end) as mens_sales
            from retail_sales
            where kind_of_business in ('Women''s clothing stores','Men''s clothing stores')
            group by 1
            order by sales_year)a
            order by sales_year;
          ```
          * without `subquery`
          * adding `where` clause filter to remove 2020, since a few months have *null values*
          ```sql
          ---PostgreSql
          select
            date_part('year',sales_month) as sales_year,
            sum(case when kind_of_business='Women''s clothing stores' then sales end)
            -
            sum(case when kind_of_business='Men''s clothing stores' then sales end )
            as womens_minus_mens
            from retail_sales
            where kind_of_business in ('Womne''s clothing stores','Men''s cloting stores')
            and sales_month<='2019-12-01'
            group by 1;
          ```
       2. **Ratio**
          ```sql
          ---PostgreSql
          select
            sales_year,
            ---using mens'sales as the baseline
            womens_sales/mens_sales as womens_times_of_mens
            from
            (select
            date_part('year',sales_month) as sales_year,
            sum(case when kind_of_business='Women''s clothing stores' then sales end) as womens_sales,
            sum(case when kind_of_business='Men''s clothing stores' then sales end) as mens_sales
            from retail_sales
            where kind_of_business in ('Women''s clothing stores','Men''s clothing stores')
            and sales_month<='2019-12-01'
            group by 1
            order by sales_year)a
			      order by sales_year;
          ```
       3. **Percent difference**
          ```sql
          ---PostgreSql
          select
            sales_year,
            (womens_sales/mens_sales-1)*100 as womens_pct_of_mens
            from
            (select
            date_part('year',sales_month) as sales_year,
            sum(case when kind_of_business='Women''s clothing stores' then sales end) as womens_sales,
            sum(case when kind_of_business='Men''s clothing stores' then sales end) as mens_sales
            from retail_sales
            where kind_of_business in ('Women''s clothing stores','Men''s clothing stores')
            and sales_month<='2019-12-01'
            group by 1
            order by sales_year)a
      			order by sales_year;
          ```

* The choice of which to use depends on the audience and the norms in each domain
* The transformation allows us to analyse time series by comparing related parts.

## Percent of Total Calculations
* To check each part's **contribution**
* Unless the data already contains a time series of the total values, need to calculate **the overall total** in order to calculate the per cent of the total for each row
* using `self-JOIN` or a `window` function
  * `self-JOIN`
    * Anytime a table is joined to **itself**
    * As long as each instance of the table in the query is given a different alias, the database will treat them all as **distinct tables**
    ```sql
    ---PostgreSql
    select
	sales_month,
	kind_of_business,
	sales*100/total_sales as pct_total_sales
	from
	(select
	a.sales_month,
	a.kind_of_business,
	a.sales,
	sum(b.sales) as total_sales
	from retail_sales a
	join retail_sales b on a.sales_month=b.sales_month
	and b.kind_of_business in ('Mens''s clothing stores','Women''s clothing stores')
	where 
	a.kind_of_business in ('Men''s clothing stores','Women''s clothing stores')
	group by 1,2,3
	order by a.sales_month) aa
	order by sales_month;
    ```
  * using `sum` window function and `PARTITION BY`
  * `PARTITION BY`: indicating the **section** of the table within which the function should calculate
	  ```sql
	  ---PostgreSql
	  select
		sales_month,
		kind_of_business,
		sales,
		sum(sales) over (partition by sales_month) as total_sales,
		sales*100/sum(sales) over (partition by sales_month) as pct_total
		from retail_sales
		where kind_of_business in ('Men''s clothing stores','Women''s clothing stores')
		order by sales_month;
	  ```

* Finding the percent of sales within **a longer time period**
  * `self-JOIN`
    ```sql
    ---PostgreSql
    select
	sales_month,
	kind_of_business,
	sales*100/yearly_salse as pct_yearly
	from
	(select
	a.sales_month,
	a.kind_of_business,
	a.sales,
	sum(b.sales) as yearly_salse
	from retail_sales a
	join retail_sales b on
	date_part('year',a.sales_month)=date_part('year',b.sales_month)
	and a.kind_of_business=b.kind_of_business
	and b.kind_of_business in ('Men''s clothing stores','Women''s clothing stores')
	where a.kind_of_business in ('Men''s clothing stores','Women''s clothing stores')
	group by 1,2,3
	order by 
	a.sales_month) aa;
    ```
  * `window function`
    ```sql
    ---PostgreSql
    select
	sales_month,
	kind_of_business,
	sales,
	sum(sales) over (partition by date_part('year',sales_month),kind_of_business) as yearly_sales,
	sales*100/sum(sales) over (partition by date_part('year',sales_month),kind_of_business) as pct_yearly
	from retail_sales
	where kind_of_business in ('Men''s clothing stores','Women''s clothing stores')
	order by sales_month;
    ```
## Indexing to See Percent Change over Time
* Understanding the changes in a time series relative to a **base period(starting point)**
* **Consumer Price Index(CPI)**
  * Tracking the change in the prices of items
  * Tracking inflation
  * Deciding on increasing the salary
  * and for many other applications
* Using `first_value` window function
  * Returning the sales value for the first row in the entire data set
    ```sql
    ---PostgreSql
    select sales_year,sales,
	first_value(sales) over (order by sales_year) as index_sales
	from
	(select date_part('year',sales_month) as sales_year,
	sum(sales) as sales
	from retail_sales
	where kind_of_business='Women''s clothing stores'
	group by 1)a;
    ```
  * Finding the **percent change**
    ```sql
    ---PostgreSql
    select
	sales_year,
	sales,
	(sales/first_value(sales) over (order by sales_year)-1)*100 as pct_from_index
	from
	(select date_part('year',sales_month) as sales_year,
	sum(sales) as sales
	from retail_sales
	where kind_of_business='Women''s clothing stores'
	group by 1)a;
    ```
