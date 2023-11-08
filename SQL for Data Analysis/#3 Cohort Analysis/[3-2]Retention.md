# Data Sets
* Legislators table
![legislators](https://github.com/Soon607/Data_Analysis_SQL/assets/98214539/97a059d0-df29-4091-a3d5-345395a5cc05)
* Legislatrs_terms table
![legislators_terms](https://github.com/Soon607/Data_Analysis_SQL/assets/98214539/7f929b68-9a54-44a1-8301-1235fecc99c7)
# Retention
* To *keep* or *continue* something
* Whether the starting size of the cohort(number of subscribers or employees, amount spent, or another key metric) will remain constant, decay, or increase over time?
* The starting size will tend to decay over time since a cohort can lose but cannot gain new members once it is formed

## SQL for a Basic Retention Curve
* Three components(the cohort definition, a time series or actions, and an aggregate metric) are needed for retention analysis.
* In this case, the cohort members will be the legislators, the time series will be the terms in office for each legislator, and the metric of interest will be the `count` of those who are still in office each period from the starting date.
* First step
  * Finding the first date each legislator took office.
    ```sql
    ---Postgresql
    select
      id_bioguide,
      min(term_start) as first_term
      from legislators_terms
      group by 1;
    ```
* Next step
  * Putting the above code into a `subquery` and `JOIN` it to the time series.
    ```sql
    ---Postgresql
    select
      date_part('year',age(b.term_start,a.first_term)) as period,
      count(distinct a.id_bioguide) as cohort_retained
      from
      (select id_bioguide, min(term_start) as first_term
      from legislators_terms
      group by 1)a
      join legislators_terms b on a.id_bioguide=b.id_bioguide
      group by 1
    ```
    * `age` function: calculating the intervals between each *term_start* and the *first_term* for each legislator
    * Applying `date_part` transforms into the number of yearly periods
* Final step
  * Calculating the total *cohort_size* and populating it in each row
  * So, the *cohort_retained* can be divided by it
    ```sql
    ---Postgresql
    select
        period,
        first_value(cohort_retained) over (order by period) as cohort_size,
        cohort_retained,
        100.0*cohort_retained/first_value(cohort_retained) over (order by period) as pct_retained
        from
        (select
        date_part('year',age(b.term_start,a.first_term)) as period,
        count(distinct a.id_bioguide) as cohort_retained
        from
        (select id_bioguide, min(term_start) as first_term
        from legislators_terms
        group by 1) a
        join legislators_terms b
        on a.id_bioguide=b.id_bioguide
        group by 1)aa
    ```
* **Pivot and flatten the cohort retention result to show it in table format**
  ```sql
  ---Postgresql
  select
      cohort_size,
      max(case when period=0 then pct_retained end) as yr0,
      max(case when period=1 then pct_retained end) as yr1,
      max(case when period=2 then pct_retained end) as yr2,
      max(case when period=3 then pct_retained end) as yr3,
      max(case when period=4 then pct_retained end) as yr4,
      max(case when period=5 then pct_retained end) as yr5
      from
      (select
      period,
      first_value(cohort_retained) over (order by period) as cohort_size,
      cohort_retained,
      100.0*cohort_retained/first_value(cohort_retained) over (order by period) as pct_retained
      from
      (select
      date_part('year',age(b.term_start,a.first_term)) as period,
      count(distinct a.id_bioguide) as cohort_retained
      from
      (select id_bioguide, min(term_start) as first_term
      from legislators_terms
      group by 1) a
      join legislators_terms b
      on a.id_bioguide=b.id_bioguide
      group by 1)aa)aaa
      group by 1
  ```
* Summary
  * Retention appears to be quite low
  * A representative's term lasts *two years* and a senator's term lasts *six years*; there are missing data for years in which a legislator was still in office but didn't start a new term
    * It led to the misunderstanding of the data
  * Options
    * Measuring only a two-or six-year cycle
    * Using a strategy that can be employed to fill in the *missing* data.
## Adjusting Time Series to Increase Retention Accuracy
* When dealing with time series data, such as in cohort analysis, it's important to **consider not only the data that is present but also whether that data accurately reflects the presence or absence of entities at each period.**
  * Deriving the span of time in which the entity is still present can help
    * with an explicit *end date*
    * with knowledge of the length of the subscription or term
* To correct the problem in the previous section, fill in the *missing values* for the years legislators are still in office between new terms.
* Calculating retention using a start and end date defined is the most accurate approach
  * In this example, I would pick *December 31* as a strategy for normalising around these varying start dates since legislators can take on their duties on different days, respectively, because of some unexpected things.
  * First step
    * Creating a data set that contains a record for each December 31 that each legislator was in office.
    ```sql
    ---Postgresql
    select
      a.id_bioguide,
      a.first_term,
      b.term_start,
      b.term_end,
      c.date,
      date_part('year',age(c.date,a.first_term)) as period
      from
      (select id_bioguide,min(term_start) as first_term
      from legislators_terms
      group by 1) a
      join legislators_terms b 
      on a.id_bioguide=b.id_bioguide
      left join date_dim c 
      on c.date between b.term_start and b.term_end
      and c.month_name='December' and c.day_of_month=31
    ```
    * The second `JOIN` to the `date_dim` retrieves the dates that fall between the start and end dates by restricting the returned values to *c.month_name='December'* and *c.day_of_month=31*
    * Even though more than 11 months may have elapsed between being sworn in in January and December 31, the first year still appears as 0
  * Next step
    * Calculating the *cohort_retained* for each period.
    ```sql
    ---Postgresql
    select
      coalesce(date_part('year',age(c.date,a.first_term)),0) as period,
      count(distinct a.id_bioguide) as cohort_retained
      from
      (select id_bioguide,min(term_start) as first_term
      from legislators_terms
      group by 1) a
      join legislators_terms b 
      on a.id_bioguide=b.id_bioguide
      left join date_dim c 
      on c.date between b.term_start and b.term_end
      and c.month_name='December' and c.day_of_month=31
      group by 1;
    ```
  * Final step
    * Calculating the *cohort_size* and *pct_retained*
    ```sql
    ---Postgresql
    select
      period,
      first_value(cohort_retained) over (order by period) as cohort_size,
      cohort_retained,
      cohort_retained*1.0/first_value(cohort_retained) over (order by period) as pct_retained
      from
      (select
      coalesce(date_part('year',age(c.date,a.first_term)),0) as period,
      count(distinct a.id_bioguide) as cohort_retained
      from
      (select id_bioguide,min(term_start) as first_term
      from legislators_terms
      group by 1) a
      join legislators_terms b 
      on a.id_bioguide=b.id_bioguide
      left join date_dim c 
      on c.date between b.term_start and b.term_end
      and c.month_name='December' and c.day_of_month=31
      group by 1)aa
    ```
* When the data set does not contain an end date
  * Option1
    * Adding a fixed interval to the start date when the length of a subscription or term is known
    ```sql
    ---Postgresql
    select
      a.id_bioguide,a.first_term,
      b.term_start,
      case
      when b.term_type='rep' then b.term_start+interval '2 years'
      when b.term_type='sen' then b.term_start+interval '6 years'
      end as term_end
      from
      (select
      id_bioguide,min(term_start) as first_term
      from legislators_terms
      group by 1)a
      join legislators_terms b on a.id_bioguide=b.id_bioguide;
    ```
    * The above code can then be plugged into the retention code to derive the *period* and *pct_retained*
    * However, it fails to capture instances in which a legislator did not complete a full term.
  * Option2
    * Using the *subsequent starting date*, *minus one day* as the *term_end_date*
    ```sql
    ---Postgresql
    select
      a.id_bioguide,
      a.first_term,
      b.term_start,
      lead(b.term_start) over (partition by a.id_bioguide order by b.term_start)-interval '1day' as term_end
      from
      (select id_bioguide,min(term_start) as first_term
      from legislators_terms
      group by 1)a
      join legislators_terms b
      on a.id_bioguide=b.id_bioguide
    ```
    * using `lead` window function
    * can then be plugged into the retention code
    * However, when there is no subsequent term, the `lead` function returns *null*
    * And, it assumes that terms are always consecutive, with no time spent out of office

## Summary
* When we make adjustments to fill in missing data, we need to be careful about the assumptions we would make.
* In subscription or term-based contexts, explicit start and end dates tend to be most accurate.
* Adding a fixed interval or setting the end date relative to the next start date can be used when no end date is present and we have a reasonable expectation that most customers or users will stay for the duration assumed

  
