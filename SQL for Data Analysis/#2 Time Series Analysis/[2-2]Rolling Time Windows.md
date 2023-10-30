# Rolling Time Windows
* Another technique for smoothing data(**Moving Calculations**)
* Taking into account **multiple periods**
* **Last Twelve Months**(LTM), **Trailing Twelve Months**(TTM), **Year-To-Date**(YTD)

## Important pieces of rolling time series calculation
1. The size of the window(The number of periods)
   * Large Windows(more time periods)
     * will have a greater smoothing effect
     * there is a risk of losing sensitivity to important short-term changes
   * Shorter windows(fewer time periods)
     * less smoothing (too little noise reduction)
     * more sensitivity to short-term changes
2. Aggregate function used
   * Moving averages are the most common
   * Moving sums, counts, min, max
3. Choosing the partitioning or grouping of the data (included in the window)

## Retail Sales Data Set
![The Retail Sales Data Set](https://github.com/Soon607/Data-Analysis-SQL-/assets/98214539/9a61ff03-145e-4ef0-8912-c268ffefd524)

## Calculating Rolling Time Windows
* Using `self-JOIN`
  * First, developing the intuition for what will go into the calculations
    ```sql
    ---Postgresql
    select
      a.sales_month,
      a.sales,
      b.sales_month as rolling_sales_month,
      b.sales as rolling_sales
      from retail_sales a
      join retail_sales b 
      on a.kind_of_business=b.kind_of_business
      and b.sales_month between a.sales_month-interval '11 months' and a.sales_month
      and b.kind_of_business='Women''s clothing stores'
      where a.kind_of_business='Women''s clothing stores'
      and a.sales_month='2019-12-01';
    ```
    * Table a: Gathering dates
    * Table b: Gathering the 12 individual months of sales(that will go into the moving average)
  * Applying the aggregation
    ```sql
    ---Postgresql
    select
      a.sales_month,
      a.sales,
      avg(b.sales) as moving_avg,
      count(b.sales) as records_count
      from retail_sales a
      join retail_sales b 
      on a.kind_of_business=b.kind_of_business
      and b.sales_month between a.sales_month-interval '11 months' and a.sales_month
      and b.kind_of_business='Women''s clothing stores'
      where a.kind_of_business='Women''s clothing stores'
      and a.sales_month>='2019-12-01'
      group by 1,2
      order by a.sales_month;
    ```
* Using `frame clause`
  * Allow to specify **which records to include in the window**
  * can be:
    * UNBOUNDED PRECEDING
    * offset PRECEDING
    * CURRENT ROW
    * offset FOLLOWING
    * UNBOUNDED FOLLOWING
  * *PRECEDING*: Including rows before the current row
  * *FOLLOWING*: Including rows that occur after the current row
  * *UNBOUNDED*: Including all records in the partition before or after the current row
  * *offset*: The number of records
    ```sql
    ---Postgresql
    select
      sales_month,
      avg(sales) over (order by sales_month rows between 11 preceding and current row) as moving_avg,
      count(sales) over (order by sales_month rows between 11 preceding and current row) as records_count
      from retail_sales
      where kind_of_business='Women''s clothing stores';
    ```
## Rolling Time Windows with Sparse Data
* Dealing with a problem when there is no record for the month(day, year)itself.
* `date dimension` can help solve the problem
  * *date dimension*: A static table that contains a row for each calendar date
    ![image](https://github.com/Soon607/Data-Analysis-SQL-/assets/98214539/b2469323-1fe2-4a2f-9b3d-49180fc2fc28)
  ```sql
  ---Postgresql
  select
    a.date,
    b.sales_month,
    b.sales
    from date_dim a
    join
    (select sales_month,sales
    from retail_sales
    where kind_of_business='Women''s clothing stores'
    and date_part('month',sales_month) in (1,7)) b
    on b.sales_month between a.date-interval '11 months' and a.date
    where a.date=a.first_day_of_month
    and a.date between '1993-01-01' and '2020-12-01';
  ```
  * Getting the moving average
    ```sql
    ---Postgresql
    select
      a.date,
      avg(b.sales) as moving_avg,
      count(b.sales) as records
      from date_dim a
      join
      (select sales_month,sales
      from retail_sales
      where kind_of_business='Women''s clothing stores'
      and date_part('month',sales_month) in (1,7)) b
      on b.sales_month between a.date-interval '11 months' and a.date
      where a.date=a.first_day_of_month
      and a.date between '1993-01-01' and '2020-12-01'
      group by 1;
    ```
      * The result set includes a row for every month.
      * However, the moving average stays constant until a new data point is added
* Without the date dimension
  * `SELECT` the `DISTINCT` dates needed and `JOIN` them in a subquery
    ```sql
    ---Postgresql
    select
      a.sales_month,
      avg(b.sales) as moving_avg
      from
      (select distinct sales_month
      from retail_sales
      where sales_month between '1993-01-01' and '2020-12-01'
      order by sales_month)a
      join retail_sales b
      on b.sales_month between a.sales_month-interval '11 months' and a.sales_month
      and b.kind_of_business='Women''s clothing stores'
      group by 1
    ```
