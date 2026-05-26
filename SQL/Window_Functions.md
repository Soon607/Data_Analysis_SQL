# CUME_DIST Function
Calculating the **cumulative distribution** of a value within a set of values (Returns the relative position of a value in a set of values)
* The syntax of the `CUME_DIST()`
  
  ```sql
  CUME_DIST() OVER (
    [PARTITION BY partition_expression, ... ]
    ORDER BY sort_expression [ASC | DESC], ...
  )
  ```
* `PARTITION BY` clause: Dividing rows into multiple partitions, on which the `CUME_DIST()` function is applied.
* `ORDER BY` clause
  * `order by asc`(Default): Calculating **the percentage of rows** that have a value **less than** or **equal to** the current row.

    * **A value close to 1.0**: This row is **near the top of the dataset.**
    * (number of rows with values <= current row value) / total number of rows in the partition.
  * `order by desc`: Calculating **the percentage of rows** that have a value **greater than** or **equal to** the current row.
    * **A value close to 1.0**: This row is **near the bottom of the dataset.**
    * (number of fows with values >= current row value) / total number of rows in the partition.
* `Return` value: A double precision value (0 < `cume_dist()` <= 1)
## Examples
* Dataset
<img width="802" height="264" alt="Image" src="https://github.com/user-attachments/assets/d15f2132-6d55-4e00-9da3-c23d6110b54e" />

     
* Using `cume_dist()
```sql
SELECT
    name,
    year,
    amount,
    CUME_DIST() OVER (
        ORDER BY amount
    )
FROM
    sales_stats
WHERE
    year = 2018;
```


* Output
<img width="1002" height="560" alt="Image" src="https://github.com/user-attachments/assets/8fe26f13-2b44-404e-afd8-7f3a73295df3" />


# PERCENT_RANK Function
Evaluating the **relative standing** of a value within a set of values
* The syntax of Percent_rank()
```sql
PERCENT_RANK() OVER (
    [PARTITION BY partition_expression, ... ]
    ORDER BY sort_expression [ASC | DESC], ...
)
```
* `PARTITION BY` Clause: Dividing rows into multiple partitions to which the function is applied.
  
* How it works
  * Percent_rank = (rank-1) / total_rows - 1
  * rank: The rank of the current row.
    
* Return: 0 < `PERCENT_RANK()` <=1
  * The first value always has a rank of 0.
  * The last row has a 1.

* Key Differences
  * **Ties**: If multiple rows have the same value in the `order by` column, they will receive the same `percent_rank.
    
  * **Nulls**: Null values are treated based on your `order by` clause. (Typically sorting last by default.)
    
  * **vs `CUME_DIST()`**: While `percent_rank` tells you the **relatvie rank (starting at 0)**, `cume_dist` tells you the **proportion of rows** less tahn or equal to the current row. 
## Examples
* Dataset


<img width="539" height="406" alt="Image" src="https://github.com/user-attachments/assets/d20ad521-f588-494c-b90a-5c9ef635fc85" />


* Using the function
```sql
SELECT
    name,
	amount,
    PERCENT_RANK() OVER (
        ORDER BY amount
    )
FROM
    sales_stats
WHERE
    year = 2019;
```


<img width="501" height="189" alt="Image" src="https://github.com/user-attachments/assets/60abb250-c39f-4dce-91d1-5a9728ac1472" />


* Using the function over a partition
```sql
SELECT
    name,
	amount,
    PERCENT_RANK() OVER (
		PARTITION BY year
        ORDER BY amount
    )
FROM
    sales_stats;
```


<img width="580" height="324" alt="Image" src="https://github.com/user-attachments/assets/de4d8a68-8f43-4d01-8f1c-c2a239cf9098" />
