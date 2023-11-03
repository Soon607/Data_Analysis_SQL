# Analysing with Seasonality
**Seasonality**: Any pattern that repeats over regular intervals (can be *predicted*)
* **Examples of Seasonality** (having different time scales)
  * The presidential election in the US happens every four years and can lead to distinct patterns in media coverage
  * School dominates Monday to Friday
  * Chores and leisure activities dominate the weekend

* **Approaches to analysing Seasonality**
  * Smoothing it out, either by aggregating the data to a less granular time or by rolling windows
  * Benchmarking against similar periods and analyse the difference

## Data Set
![The Retail Sales Data Set](https://github.com/Soon607/Data-Analysis-SQL-/assets/98214539/9a61ff03-145e-4ef0-8912-c268ffefd524)

## Period-over-Period Comparisons: YoY and MoM and DoD
* Comparing a time-period to the previous value in the series
* The comparison might be (depending on the level of aggregation)
  * year-over-year(YoY)
  * month-over-month(MoM)
  * day-over-day(DoD)

* Using `lag` function
  * `lag` function
    * returning a previous or lagging value from a series
    * return value can be any field and any data type
    * The optional *OFFSET* indicates how many rows back in the partition to take the *return_value* (default is 1)
    * is calculated over a partition, with sorting by the `ORDER BY`
  * Calculating **MoM** and **YoY**
    * Developing intuition by using `lag`
      ```sql
      ---Postgresql
      select kind_of_business,sales_month,sales,
          lag(sales_month) over (partition by kind_of_business order by sales_month) as prev_month,
          lag(sales) over (partition by kind_of_business order by sales_month) as prev_month_sales
          from retail_sales
          where kind_of_business='Book stores';
      ```
    * Calculating the **percent change** from the previous value (MoM)
      ```sql
      ---Postgresql
      select
          kind_of_business,sales_month,sales,
          (sales/lag(sales) over (partition by kind_of_business order by sales_month)-1)*100 as pct_growth_from_previous
          from retail_sales
          where kind_of_business='Book stores';
      ```
    * Calculating the YoY comparison
      ```sql
      ---Postgresql
      select sales_year,yearly_sales,
          lag(yearly_sales) over (order by sales_year) as prev_year_sales,
          (yearly_sales/lag(yearly_sales) over (order by sales_year)-1)*100 as pct_growth_from_previous
          from
          (select date_part('year',sales_month) as sales_year,
          sum(sales) as yearly_sales
          from retail_sales
          where kind_of_business='Book stores'
          group by 1) a
      ```
* Period-over-period calculations don't quite allow for seasonality analysis in the data set.
* Comparing current values to the values for the same month in the previous year can tackle this problem.
## Period-over-Period Comparisons: Same Month Versus Last Year
* Comparing data for one time period to data for a similar previous time period can be a useful way to control for the seasonality
* Comparing monthly sales to the sales for the same month in the previous year
  * Looking up the value for the matching month number from the prior year.
    ```sql
    ---Postgresql
    select
      sales_month,
      sales,
      lag(sales_month) over (partition by date_part('month',sales_month) order by sales_month) as prev_year_month,
      lag(sales) over (partition by date_part('month',sales_month) order by sales_month) as prev_year_sales
      from retail_sales
      where kind_of_business='Book stores'
    ```
    * The first `lag` returns the same month for the prior year
  * Calculating absolute difference and percent change from previous
    ```sql
    ---Postgresql
    select
      sales_month,
      sales,
      sales-lag(sales) over (partition by date_part('month',sales_month) order by sales_month) as absolute_diff,
      (sales/lag(sales) over (partition by date_part('month',sales_month) order by sales_month)-1)*100 as pct_diff
      from retail_sales
      where kind_of_business='Book stores'
    ```
* Creating a graph that lines up the same time period
  * In this case, months with a line for each time series.
  * creating a result set (row for each month/column for each of the years)
    ```sql
    ---Postgresql
    select
      date_part('month',sales_month) as month_number,
      to_char(sales_month,'Month') as month_name,
      max(case when date_part('year',sales_month)=1992 then sales end) as sales_1992,
      max(case when date_part('year',sales_month)=1993 then sales end) as sales_1993,
      max(case when date_part('year',sales_month)=1994 then sales end) as sales_1994
      from retail_sales
      where kind_of_business='Book stores'
      and sales_month between '1992-01-01' and '1994-12-01'
      group by 1,2
    ```
## Comparing to Multiple Prior Periods
* A useful way to reduce the noise that arises from seasonality
* When a prior period was impacted by unusual events, compared to a single prior period is insufficient.
  * one of them was a holiday from a current day and the previous day
  * economic events, severe weather, a site outage
* **Comparing current values to an aggregate of multiple prior periods can help smooth out these fluctuation**
* Using the `lag` function
  * should take advantage of the optional offset value
    ```sql
    ---Postgresql
    select sales_month,sales,
      lag(sales) over (partition by date_part('month',sales_month) order by sales_month) as prev_sales_1,
      lag(sales,2) over (partition by date_part('month',sales_month) order by sales_month) as prev_sales_2,
      lag(sales,3) over (partition by date_part('month',sales_month) order by sales_month) as prev_sales_3
      from retail_sales
      where kind_of_business='Book stores'
    ```
    * An offset value of 2 and 3 returns the value from the 2,3 previous rows, respectively.
  * Calculating the per cent of the rolling average of three prior periods.
    ```sql
    ---Postgresql
    select
      sales_month,
      sales,
      sales/((prev_sales_1+prev_sales_2+prev_sales_3)/3) as pct_of_3_prev
      from
      (select sales_month,sales,
      lag(sales) over (partition by date_part('month',sales_month) order by sales_month) as prev_sales_1,
      lag(sales,2) over (partition by date_part('month',sales_month) order by sales_month) as prev_sales_2,
      lag(sales,3) over (partition by date_part('month',sales_month) order by sales_month) as prev_sales_3
      from retail_sales
      where kind_of_business='Book stores')a
    ```
* Using `frame` clause
  ```sql
  ---Postgresql
  select
    sales_month,
    sales,
    (sales/avg(sales) over (partition by date_part('month',sales_month) order by sales_month rows between 3 preceding and 1 preceding)*100) as pct_of_prev_3
    from retail_sales
    where kind_of_business='Book stores'
  ```
* **Comparing data points against multiple prior time periods can give us an even smoother trend to compare and to determine what is actually happening in the current time period.**
