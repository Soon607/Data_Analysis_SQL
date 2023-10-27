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

### Cleaning Data with CASE Transformations

1. Standardising the values
```sql
---PostgreSQL
case when gender='F' then 'Female'
     when gender='female' then 'Female'
     when gender='femme' then 'Female'
     else gender
     end as gender_cleaned
```
2. Categorization(Enrichment)
   ```sql
   ---PostgreSQL
   select response_id,likelihood,
   case when likelihood<=6 then 'Detractor'
        when likelihood<=8 then 'Passive'
        else 'Promoter'
        end as response_type
   from nps_responses
   ```
   * When the input isn't continuous or when values in order shouldn't be grouped (using `IN` list)
     ```sql
     ---PostgreSQL
     case when likelihood in (0,1,2,3,4,5,6) then 'Detractor'
          when likelihood in (7,8) then 'Passive'
          when likelihood in (9,10) then 'Promoter'
          end as response_type
     ```
   * `CASE` statement can consider multiple columns and can contain `AND/OR` logic
   * They can also be nested, though often this can be avoided with `AND/OR` logic
     ```sql
     ---PostgreSQL
     case when likelihood <=6 and country='US' and high_value=true then 'US high value detractor'
          when likelihood >=9 and (country in ('CA','JP') or high_value=true) then 'some other label'
     ....end
     ```
3. Creating flags
   * Indicating whether a certain value is present without returning the actual value
     * (useful during profiling for understanding how common the existence of a particular attribute is
   * Preparation of a data set for statistical analysis
     * A dummy variable(taking a value of 0 or 1): Indicating the presence or absence of some qualitative variable
   ```sql
   ---PostgreSQL
   select
   customer_id,
   case when gender='F' then 1 else 0 end as is_female,
   case when likelihood in (9,10) then 1 else 0 end as is_promoter,
   from...;
   ```
4. Flattening the Data
   * Useful when working with a data set that has multiple rows per entity (ex: line items in an order)
   * Wrapping in an aggregate and turn it into a flag at the same time by using **1(True)** and **0(False)** as the return value
   ``` sql
   ---PostgreSQL
   select
   customer_id,
   max(case when fruit='apple' then 1 else 0 end) as bought_apples,
   max(case when fruit='orange' then 1 else 0 end) as bought_oranges
   from....
   group by 1;
   ```
     * For each customer, the `CASE` statement returns 1 for any row with a fruit type of 'apple'.
     * The `max` is evaluated and will return **the largest value** from any of the rows

### Type Conversions and Casting
* **Type conversion functions** allow pieces of data with the appropriate format to be changed from one data type to another
1. `cast` function
   * **cast(input as data_type) or input::data_type**
     ```sql
     ---PostgreSQL
     cast(1234 as varchar)
     1234::varchar
     ```
2. Dealing with **Dates** and **Datetimes**
   * Casting the **TIMESTAMP** to a **DATE**
     ```sql
     ---PostgreSQL
     select tx_timestamp::date,count(transaction) as num_transactions
     from ....
     group by 1;
     ```
   * Casting the **Date** to a **Timestamp**
     ```sql
     ---PostgreSQL
     select tx_date::timestamp,count(transaction) as num_transactions
     from ....
     group by 1
     ```
3. Converting between string values and dates
   ```sql
   ---PostgreSQL
   (year||','||month||'-'||day)::date
   ---or
   cast(concat(year,'-',month,'-',day) as date)
   ```
   * using `date` function
     ```sql
     ---PostgreSQL
     date(concat(year,'-',month,'-',day))
     ```

### Dealing with Nulls(coalesce,nullif Functions)
* Nulls: representing fields for which no data was collected or that are not applicable to that row
* A variety of unexpected and frustrating results can be output from queries if we do not care about nulls
* Ways of dealing with nulls
  1. Allowing nulls
  2. Rejecting nulls
  3. Populating a default value
 * Ways to replace with alternative values
   1. `Case` statements: Quite complicated
   2. `coalesce`: A more compact way, taking two or more arguments and returning the first one that is not null
      ```sql
      ---Postgresql
      coalesce(num_orders,0)
      coalesce(address,'Unknown')
      coalesce(column_a,column_b)
      coalesce(column_a,column_b,column_c)
      ```
   3 `nullif`: comparing two numbers, and if they **are not** equal, it returns the number, if they are **equal**, the function returns null
     ```sql
     ---PostgreSQL
     nullif(6,7)
     ---return 6
     nullif(6,6)
     ---return null
     ```
     * useful for turning values back into nulls when knowing a **certain default value**

### Missing Data
* It doesn't always need to be fixed or filled
* It can reveal the underlying system design or biases in the data collection process

#### Imputation techniques
* for filling in missing data
* including filling with an average or median of the data set, or with the previous value

1. **Filling with a constant value**
   * useful when the value is known for some records even though they were not populated in the database
   * Using a `CASE` statement
     ```sql
     ---PostgreSQL
     case when price is null and item_name='xyz' then 20 else price end as price
     ```
2. **Filling with a derived value**
   * `CASE` statement
   * `Mathematical function` on other columns
     ```sql
     ---PostgreSQL
     select gross_sales-discount as net_sales...
     ```
     * Assuming losing a field for the `net_sales` amount for each transaction.
     * But, having the `gross_sales` and `discount` fields populated
     * can calculate `net_sales` by subtracting `discount` from `gross_sales`

3. **Filling with values from other rows in the data set**
   * **Fill forward**: Carrying over a value from the previous row (`lag` function)
   * **Fill Back**: Using a value from the next row
       ```sql
       ---PostgreSQL
       lag(product_price) over(partition by product order by order_date)
       lead(product_price) over(partition by product order by order_date)
       ```
4. **Taking the average values of fields**

## Preparing(Shaping Data)
Manipulating the way the data is represented in columns and rows

* **Important concepts in shaping data**
  1. Figuring out the granularity of data
     * Data have varying levels of detail
  2. Flattering data
     * Reducing the number of rows that represent an entity, including down to a single row
     * Joining multiple tables together to create a single output data set
### Pivoting with CASE statements
* A pivot table
  * Summarising data sets by arranging the data into rows, according to the values of an attribute, and columns, according to the values of another attribute
  * Reshaping the data into a more compact and easily understandable form
    ```sql
    ---PostgreSQL
    select order_date,
    sum(case when product='shirt' then order_amount else 0 end) as shirts_amount,
    sum(case when product='shoes' then order_amount else 0 end) as shoes_amount,
    sum(case when product='hat' then order_amount else 0 end) as hats_amount
    from orders
    group by 1
    ;
    ```
    * with the `sum` aggregation, using **else 0** is optional in case there are nulls
    * However, when using `count` or `count distinct`, do not include `else` statment
      * It would inflate the result set (database won't count a null, but it will count a substitute value such as zero)
     
### Unpivoting with UNION Statements
* When we need to **move data stored in columns into rows instead to create tidy data**
  ```sql
  ---PostgreSQL
  select country,
  '1980 as year,
  year_1980 as population
  from country_populations
  union all
  select country,
  '1990' as year,
  year_1990 as population
  from country_populations
  ......from country_populations;
  ```
