# Cross-Section Analysis, Through a Cohort Lens
* Challenges with cohort analysis
  * Even as they make changes within cohorts easy to spot, it can be difficult to spot changes in the overall composition of a customer or user base.
  * **Mix shifts** can also occur, making later cohorts different from earlier ones.
    * Mix shifts
      * Changes in the composition of the customer or user base over time
      * can occur due to international expansion, shifting between organic and paid acquisition strategies....
  * Creating additional cohorts, or segments, along any of these suspected lines can help diagnose *whether a mix shift is happening*
*****
* Cohort analysis can be contrasted with *cross-sectional analysis*
  * Cross-sectional analysis: Comparing individuals or groups at a single point in time.
  * For example, Cross-sectional analysis can correlate years of education with current income.
  * The positive side of collecting data sets for cross-sectional analysis
    * no time series is necessary
    * Cross-sectional analysis can be insightful, generating hypotheses for further investigation
  * The negative side
    * A form of selection bias(**survivorship**) usually exists, which can lead to false conclusion.

## Survivorship Bias
* The logical error of focusing on the people or things that made it past some selection process while ignoring those that did not.
* Commonly, this is because the entities no longer exist in the data set at the time of selection because they have failed, churned, or left the population for some other reason.
* Concentrating only on the remaining population can lead to overly optimistic conclusions because failures are ignored.

## Cohort analysis helps distinguish and reduce survivorship bias
* Cohort analysis is a way to overcome survivorship bias by including all members of a starting cohort in the analysis
* We can take a series of cross-sections from a cohort analysis to understand how the mix of entities may have changed over time

## Example: Creating a time series of the share of legislators from each cohort for each year
* **Cohorting on *first_term***
* First step: Finding the number of legislators in office
  ```sql
  ---Postgresql
  select
    b.date,
    count(distinct a.id_bioguide) as legislators
    from legislators_terms a
    join date_dim b
    on b.date between a.term_start and a.term_end
    and b.month_name='December' and b.day_of_month=31
    and b.year<=2019
    group by 1
  ```
  * `JOIN` the *legislators_terms* to the *date_dim*
  * `WHERE`: the date from the *date_dim* is between start and end dates of each term
  * Using December 31 for each year to find the legislators in office at each year's end
* Adding in the century cohorting criteria
  ```sql
  ---Postgresql
  select
      b.date,
      date_part('century',c.first_term) as century,
      count(distinct a.id_bioguide) as legislators
      from legislators_terms a
      join date_dim b
      on b.date between a.term_start and a.term_end
      and b.month_name='December' and b.day_of_month=31
      and b.year<=2019
      join
      (
      select id_bioguide,min(term_start) as first_term
      from legislators_terms
      group by 1)c
      on a.id_bioguide=c.id_bioguide
      group by 1,2
  ```
  * `JOIN` to a subquery with the *first_term* calculated
* Calculating the percent of total legislators in each year that the century cohort represents.
* It can be done in a couple of ways, depending on the shape of the desired
  * First: Keeping a row for each *date* and *century* combination and use a `sum` window function in the denominator of the percentage calculation.
    ```sql
    ---Postgresql
    select
       date,
       century,
       legislators,
       sum(legislators) over (partition by date) as cohort,
       1.0*legislators/sum(legislators) over (partition by date) as pct_century
       from
       (select
       b.date,
       date_part('century',c.first_term) as century,
       count(distinct a.id_bioguide) as legislators
       from legislators_terms a
       join date_dim b
       on b.date between a.term_start and a.term_end
       and b.month_name='December' and b.day_of_month=31
       and b.year<=2019
       join
       (select id_bioguide,min(term_start) as first_term
       from legislators_terms
       group by 1)c
       on a.id_bioguide=c.id_bioguide
       group by 1,2)aa
    ```
  * Second: Resulting in one row per year, with a column for each century.
  * It may be easier to scan for trends.
    ```sql
    ---Postgresql
    select
      date,
      1.0*coalesce(sum(case when century=18 then legislators end)/sum(legislators),0) as pct_18,
      1.0*coalesce(sum(case when century=19 then legislators end)/sum(legislators),0) as pct_19,
      1.0*coalesce(sum(case when century=20 then legislators end)/sum(legislators),0) as pct_20,
      1.0*coalesce(sum(case when century=21 then legislators end)/sum(legislators),0) as pct_21
      from
      (select
      b.date,
      date_part('century',c.first_term) as century,
      count(distinct a.id_bioguide) as legislators
      from legislators_terms a
      join date_dim b
      on b.date between a.term_start and a.term_end
      and b.month_name='December' and b.day_of_month=31
      and b.year<=2019
      join
      (
      select id_bioguide,min(term_start) as first_term
      from legislators_terms
      group by 1)c
      on a.id_bioguide=c.id_bioguide
      group by 1,2)aa
      group by 1
    ```
*****
* **Cohorting on tenure**
* Finding the share of customers who are relatively new, are of medium tenure, or are long-term customers at various points in time can be insightful.
* First, calculate, for each year, the cumulative number of years in office for each legislator.
  * Since there can be gaps between terms when legislators are voted out or leave office for other reasons, we'll first find each year where the legislator was in office at the end of the year, in subquery
    ```sql
    ---Postgresql
    select
      id_bioguide,date,
      count(date) over (partition by id_bioguide order by date rows between unbounded preceding and current row) as cume_years
      from
      (select
      distinct a.id_bioguide,b.date
      from legislators_terms a
      join date_dim b
      on b.date between a.term_start and a.term_end
      and b.month_name='December' and b.day_of_month=31
      and b.year<=2019)aa
    ```
    * Used a `count` window function, with the window covering the rows `unbounded preceding` and `current row`
* Count the number of legislators for each combination of *date* and *cume_years* to create distribution.
  ```sql
  ---Postgresql
  select
      date,
      cume_years,
      count(distinct id_bioguide) as legislators
      from
      (select
      id_bioguide,date,
      count(date) over (partition by id_bioguide order by date rows between unbounded preceding and current row) as cume_years
      from
      (select
      distinct a.id_bioguide,b.date
      from legislators_terms a
      join date_dim b
      on b.date between a.term_start and a.term_end
      and b.month_name='December' and b.day_of_month=31
      and b.year<=2019)aa)aaa
      group by 1,2
  ```
* Considering grouping tenures before calculating the percentage for each tenure per year and adjusting the percentage format
  ```sql
  ---Postgresql
  select
    date,count(*) as tenures
    from
    (select
    date,
    cume_years,
    count(distinct id_bioguide) as legislators
    from
    (select
    id_bioguide,date,
    count(date) over (partition by id_bioguide order by date rows between unbounded preceding and current row) as cume_years
    from
    (select
    distinct a.id_bioguide,b.date
    from legislators_terms a
    join date_dim b
    on b.date between a.term_start and a.term_end
    and b.month_name='December' and b.day_of_month=31
    and b.year<=2019)aa)aaa
    group by 1,2)aaaa
    group by 1
    order by date
  ```
  * shows how many many tenures are represented for each year.
* Grouping the values
  * There is no single right way to group tenures.
  * In this example, I will group tenures into four cohorts (less than or equal to 4, 5 to 10, 11 to 20, equal to or more than 21)
    ```sql
    ---Postgresql
    select
        date,tenure,
        1.0*legislators/sum(legislators) over (partition by date) as pct_legislators
        from
        (select
        date,
        case when cume_years<=4 then '1 to 4'
        when cume_years<=10 then '5 to 10'
        when cume_years<=20 then '11 to 20'
        else '21+' end as tenure,
        count(distinct id_bioguide) as legislators
        from
        (select
        id_bioguide,date,
        count(date) over (partition by id_bioguide order by date rows between unbounded preceding and current row) as cume_years
        from
        (select
        distinct a.id_bioguide,
        b.date
        from legislators_terms a
        join date_dim b
        on b.date between a.term_start and a.term_end
        and b.month_name='December' and b.day_of_month=12
        and b.year<=2019
        group by 1,2)a)aa
        group by 1,2)aaa
    ```
## Summary 
* A cross-section of a population at any point in time is made up of members from multiple cohorts
* Creating a time series of these cross-sections is another interesting way of analysing trends.
* Combining this with insights from retention can provide a more robust picture of trends in any organisations

# Conclusion
* Cohort analysis is a useful way to investigate how groups change over time
  * Whether it be from the perspective of retention, repeat behaviour, cumulative actions
* Cohort analysis is retrospective, looking back at populations using intrinsic attributes or attributes derived from behaviours.
            
