# Trending the Data
* Looking for trends in the data when dealing with series data
* Trend: The direction in which the data is moving(moving up, increasing over time, moving down, decreasing over time)

## The Retail Sales Data Set
<img src="이미지 URL">
<img src="이미지 URL" width="가로 사이즈" height="세로 사이즈">

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
group by 1,2,3
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

       
               
