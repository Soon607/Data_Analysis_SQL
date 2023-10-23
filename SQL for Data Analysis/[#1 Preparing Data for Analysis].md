# Preparing Data for Analysis
* * *

## Types of Data
* Database Data Types(string,numeric,boolean,date time)
* Structured vs Unstructured
* Quantitative vs Qualitative Data
* First-, Second, and Third-Party Data
   * First-party data: collected by the organization itself.
   * Second-party data: which comes from vendors that provide a service or perform a business function on the organisation's behalf.
   * Third-party data: data that may be purchased or obtained from free sources such as those published by the government.
* Sparse data: A small amount of information within a larger set of empty or unimportant information.

## Profiling(Distributions)
* The first thing to do when starting working with any new data set.
* Checking what domain or domains are covered
* Having knowledge of how the data was generated
* Checking for how history is represented

### Distributions
Distribution allows us to understand the range of values that exist in the data and how often they occur, whether there are nulls and whether negative values exist alongside positive ones.

#### Histograms and Frequencies
* Checking the frequency of values in each field (To know a data set, particular fields within the data set)
* Useful when having a question about
    1. Whether certain values are possible
    2. If spotting an unexpected value and want to know how commonly it occurs
* Frequency checks can be done on any data type(strings, numerics, dates, and booleans)
* Frequency queries are a great way to detect sparse data
* Frequency Query(`PostgreSQL`)
```sql
select fruit, count(*) as quantity
from fruit_inventory
group by 1
```

* The number of rows can be found with `count(*)`,and the profiled field is in the `GROUP BY`.

##### Frequency plot & Histogram
* Frequency plot: To visualise the number of times something occurs in the data set(categorical data)
* Histogram: To visualise the distribution of numerical values in the dataset

#### Binning
* Binning is useful when working with continuous values
* Ranges of values are grouped together, and these groups are called **bins** or **buckets**
* The number of records that fall into each interval is then counted.
* **Bins** or **Buckets** can be variable in size or have a fixed size
* **Bins** or **Buckets** are roughly equal width, or contain roughly equal numbers of records.
* **Bins** can be created with `CASE` statements, rounding, and logarithms
    * `CASE` statements: Allowing for conditional logic to be evaluated
    * `PostgreSQL`
      ```sql
      case when condition1 then return_value_1
           when condition2 then return_value_2
           ....
           else return_value_default
           end
      ```
        * The `WHEN` condition can be an equality, inequality, or other logical condition
        * The 'THEN' return value can be a constant, an expression, or a field in the table
        * 'ELSE' is optional, and if it is not included, any nonmatches will return **null**
    * The `CASE` statement is a flexible way to control the number of bins, the range of values that fall into each bin, and how the bins are named.
    * `PostgreSQL`
      ```sql
      select
      case when order_amount<=100 then 'up to 100'
           when order_amount<=500 then '100-500'
           else '500+' end as amount_bin,
      case when order_amount<=100 then 'small'
           when order_amount<=500 then 'medium'
           else 'large' end as amount_category
      ,count(customer_id) as customers
      from orders
      group by 1,2
      ```
* Arbitrary-sized can be useful, but at other times, bins of **fixed size** are more appropriate for the analysis
* Fixed-size bins can be accomplished in a few ways, including with rounding, logarithms, and n-tiles
    * Rounding: Reducing the number of decimal places or removing them altogether by rounding to the nearest integer
        * Use `round` function
        * `PostgreSQL`
      ```sql
      round(value,number_of_decimal_places)
      ```
    * Logarithms: Creating bins, particularly in data sets in which the large values are orders of magnitude greater than the samllest values
        * Use `log` function
        * `PostgreSQL`
      ```sql
      select log(sales) as bin
      ,count(customer_id) as customers
      from table
      group by 1
      ;
      ```
#### n-Tiles
* n-tiles allow us to calculate any percentile of the data set.
* Many databases have a `median` function built in but rely on more generic **n-tile function** for the rest.
* These functions are window functions, computing across a range of rows to return a value for a single row
* They take an argument that specifies the number of bins to split the data into and, optionally, a `PARTITION BY` and `ORDER BY` clause
``` sql
ntile(num_bins) over (partition by ... order by....)
```


* A related function is `percent_rank`
* The function returns the **percentile**
* It takes no argument but requires parentheses and optionally takes a `PARTITION BY`,`ORDER BY` clause
```sql
percent_rank() over(partition by...order by....)
```

## Profiling(Data Quality)
* Data quality is critical when it comes to creating good analysis
* Profiling is a way to uncover data quality issues early on.
  * Revealing nulls, categorical codings that need to be deciphered, fields with multiple values that need to be parsed, and unusual datetime formats.

### Detecting Duplicates
* Inspecting a sample, with all columns ordered
  ``` sql
  ---PostgreSQL
  select column_a,column_b,column_c...
  from table
  order by 1,2,3
  ```
  * This will reveal whether the data is full of duplicates.
  * should try to spot duplicates by scrolling through the data


* A more systematic way to find duplicates is to `SELECT` the columns and then `count` the rows
  ```sql
  ---PostgreSQL
  select count(*)
  from
  (select column_a,column,b,column_c,...,count(*) as records from...
  group by 1,2,3...)
  where records>1
  ;
  ```
  * This will tell whether there are any cases of duplicates.
  * If the query returns 0, it is good to go
 
* An alternative to a subquery (using `HAVING` clause)
  ```sql
  ---PostgreSQL
  select column_a,column_b,column_c...,count(*) as records
  from...
  group by 1,2,3
  having count(*)>1
  ;
  ```
* For full detail on which records have duplicates, can list out all the fields and then use this information to chase down which records are problematic
  ```sql
  ---PostgreSQL
  select*
  from
  (select column_a,column_b,column_c...,count(*) as records from...
  group by 1,2,3...) a
  where records=2;
  ```
### Deduplication with Group BY and Distinct

* Removing duplicates by using the keyword `DISTINCT`
  ```sql
  ---PostgreSQL
  ---removing duplicates in the customer_id column
  select
  distinct a.customer_id,a.customer_name,a.customer_email
  from customers a
  join transactions b on a.customer_id=b.customer_id;
  ```

* Other option: using a `Group by`
  ```sql
  ---PostgreSQL
  select a.customer_id,a.customer_name,a.customer_email
  from customers a
  join transactions b on a.customer_id=b.customer_id
  group by 1,2,3
  ;
  ```

* Another way: performing an aggregation than returns one row per entity
  ```sql
  ---PostgreSQL
  select
  customer_id,
  min(transaction_date) as first_transaction_date,
  max(transaction_date) as last_transaction_date,
  count(*) as total_orders
  from table
  group by customer_id;
  ```
  * Example) If we have a number of transactions by the same customer and need to return one record per customer, we could find the `min`(first) and/or the `max`(most recent) **transaction_date**
 
## Preparing(Data Cleaning)
