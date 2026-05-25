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
![Dataset](<img width="802" height="264" alt="Image" src="https://github.com/user-attachments/assets/d15f2132-6d55-4e00-9da3-c23d6110b54e" />)          
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
![Dataset](<img width="1002" height="560" alt="Image" src="https://github.com/user-attachments/assets/8fe26f13-2b44-404e-afd8-7f3a73295df3" />)

