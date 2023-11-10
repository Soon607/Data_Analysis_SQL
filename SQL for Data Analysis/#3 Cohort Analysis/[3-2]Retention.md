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

### Summary
* When we make adjustments to fill in missing data, we need to be careful about the assumptions we would make.
* In subscription or term-based contexts, explicit start and end dates tend to be most accurate.
* Adding a fixed interval or setting the end date relative to the next start date can be used when no end date is present, and we have a reasonable expectation that most customers or users will stay for the duration assumed

## Cohorts Derived from the Time Series Itself
* After calculating retention, we can start to split the entities into cohorts

### Time-based cohorts
* Creating the cohorts based on the first or minimum date or time that the entity appears in the time series (Time-based cohorts)
  * Only one table is necessary for the cohort retention analysis(The times series itself)
  * The way is interesting because groups often start at different times and behave differently
  * can be grouped by any time granularity that is meaningful to the organisation(weekly, monthly, or yearly)
* First Example: Does the era where a legislator first took office have any correlation with their retention? (Political trends and the public mood do change over time?)
  * Using *yearly cohorts* and demonstrating swapping in centuries
  * First, adding the year of the *first term* calculated previously
    ```sql
    ---Postgresql
    select
      date_part('year',a.first_term) as first_year,
      coalesce(date_part('year',age(c.date,a.first_term)),0) as period,
      count(distinct a.id_bioguide) as cohort_retained
      from
      (select id_bioguide, min(term_start) as first_term
      from legislators_terms
      group by 1) a
      join legislators_terms b on a.id_bioguide=b.id_bioguide
      left join date_dim c
      on c.date between b.term_start and b.term_end
      and c.month_name='December' and c.day_of_month=31
      group by 1,2
    ```
  * The above query is used as the *subquery* and calculating the *cohort_size* and *pct_retained*
    ```sql
    ---Postgresql
    select
      first_year,period,
      first_value(cohort_retained) over (partition by first_year order by period) as cohort_size,
      cohort_retained,
      1.0*cohort_retained/first_value(cohort_retained) over (partition by first_year order by period) as pct_retained
      from
      (select
      date_part('year',a.first_term) as first_year,
      coalesce(date_part('year',age(c.date,a.first_term)),0) as period,
      count(distinct a.id_bioguide) as cohort_retained
      from
      (select id_bioguide, min(term_start) as first_term
      from legislators_terms
      group by 1) a
      join legislators_terms b on a.id_bioguide=b.id_bioguide
      left join date_dim c
      on c.date between b.term_start and b.term_end
      and c.month_name='December' and c.day_of_month=31
      group by 1,2)aa
    ```
  * Looking at a less granular interval and cohort the legislators by the century of the *first_term*
    ```sql
    ---Postgresql
    select
      first_year,period,
      first_value(cohort_retained) over (partition by first_year order by period) as cohort_size,
      cohort_retained,
      1.0*cohort_retained/first_value(cohort_retained) over (partition by first_year order by period) as pct_retained
      from
      (select
      date_part('century',a.first_term) as first_year,
      coalesce(date_part('year',age(c.date,a.first_term)),0) as period,
      count(distinct a.id_bioguide) as cohort_retained
      from
      (select id_bioguide, min(term_start) as first_term
      from legislators_terms
      group by 1) a
      join legislators_terms b on a.id_bioguide=b.id_bioguide
      left join date_dim c
      on c.date between b.term_start and b.term_end
      and c.month_name='December' and c.day_of_month=31
      group by 1,2)aa
    ```
    * Substituting `century` for `year` in the `date_part` function in *subquery aa*
* Cohorts can be defined from other attributes in a time series besides the first date, with options depending on the values in the table
* Using *state* field as cohorts
  * Finding the first state for each legislators
    ```sql
    ---Postgresql
    select
     distinct id_bioguide,
     min(term_start) over (partition by id_bioguide) as first_term,
     first_value(state) over (partition by id_bioguide order by term_start) as first_state
     from legislators_terms;
    ```
  * Plugging this code into the retention code
    ```sql
    ---Postgresql
    select
     first_state,period,
     first_value(cohort_retained) over (partition by first_state order by period) as cohort_size
     ,cohort_retained,
     1.0*cohort_retained/
     first_value(cohort_retained) over (partition by first_state order by period) as pct_retained 
     from
     (select
     a.first_state as first_state,
     coalesce(date_part('year',age(c.date,a.first_term)),0) as period,
     count(distinct a.id_bioguide) as cohort_retained
     from
     (select
     distinct id_bioguide,
     min(term_start) over (partition by id_bioguide) as first_term,
     first_value(state) over (partition by id_bioguide order by term_start) as first_state
     from legislators_terms)a
     join legislators_terms b
     on a.id_bioguide=b.id_bioguide
     left join date_dim c
     on c.date between b.term_start and b.term_end and
     c.month_name='December' and c.day_of_month=31
     group by 1,2)
     aa
    ```
### Summary
* Defining cohorts from the time series is relatively straightforward, using a `min` date for each entity and then converting that date into a month, year, or century as appropriate for the analysis.
* Switching between month and year or other levels of granularity is also straightforward.
* Allowing for multiple options to be tested in order to find a grouping that is meaningful for the organisations.
## Defining the Cohort from a Separate Table
* The characteristics that define a cohort exist in a table separate from the one that contains the time series
* Example: Considering whether the gender of the legislator has any impact on their retention
  * `JOIN` the legislators table that contains *gender* field to the calculation of cohort_retained table that is made previously
    ```sql
    ---Postgresql
    select
     d.gender,
     coalesce(date_part('year',age(c.date,a.first_term)),0) as period,
     count(distinct a.id_bioguide) as cohort_retained
     from
     (select id_bioguide, min(term_start) as first_term
     from legislators_terms
     group by 1)a
     join legislators_terms b 
     on a.id_bioguide=b.id_bioguide
     left join date_dim c
     on c.date between b.term_start and b.term_end
     and c.month_name='December' and c.day_of_month=31
     join legislators d on a.id_bioguide=d.id_bioguide
     group by 1,2;
    ```
  * Calculating *pct_retained*
    ```sql
    ---Postgresql
    select gender,period,
        first_value(cohort_retained) over (partition by gender order by period) as cohort_size,
        cohort_retained,
        1.0*cohort_retained/
        first_value(cohort_retained) over (partition by gender order by period) as pct_retained
        from
        (select
        d.gender,
        coalesce(date_part('year',age(c.date,a.first_term)),0) as period,
        count(distinct a.id_bioguide) as cohort_retained
        from
        (select id_bioguide, min(term_start) as first_term
        from legislators_terms
        group by 1)a
        join legislators_terms b 
        on a.id_bioguide=b.id_bioguide
        left join date_dim c
        on c.date between b.term_start and b.term_end
        and c.month_name='December' and c.day_of_month=31
        join legislators d on a.id_bioguide=d.id_bioguide
        group by 1,2)aa
    ```
* Example(2): Making a more fair comparison by restricting legislators whose *first_term* started since there have been women in Congress
  * Adding a `WHERE` filter to *subquery aa*
    ```sql
    ---Postgresql
    select gender,period,
       first_value(cohort_retained) over (partition by gender order by period) as cohort_size,
       cohort_retained,
       1.0*cohort_retained/
       first_value(cohort_retained) over (partition by gender order by period) as pct_retained
       from
       (select
       d.gender,
       coalesce(date_part('year',age(c.date,a.first_term)),0) as period,
       count(distinct a.id_bioguide) as cohort_retained
       from
       (select id_bioguide, min(term_start) as first_term
       from legislators_terms
       group by 1)a
       join legislators_terms b 
       on a.id_bioguide=b.id_bioguide
       left join date_dim c
       on c.date between b.term_start and b.term_end
       and c.month_name='December' and c.day_of_month=31
       join legislators d on a.id_bioguide=d.id_bioguide
       where a.first_term between '1917-01-01' and '1999-12-31'
       group by 1,2)aa
    ```
  * To further improve this analysis, we could cohort by both starting year or decade and gender in order to control for additional changes in retention through the 20th century and into the 21st century
### Summary
* Cohorts can be defined in multiple ways, from the time series and from other tables
* Multiple criteria, such as starting year and gender, can be used
* However, when dividing populations into cohorts based on multiple criteria, that can lead to sparse cohorts, where some of the defined groups are too small and are not represented
## Dealing with Sparse Cohorts
* If the cohort becomes small, it can be represented only sporadically in the data.
* The cohort may disappear from the result set when we would prefer it to appear with a zero retention value.
* This problem is called **sparse cohorts**, and it can be worked around with the careful use of `Left-Join.`
* Example: Cohorting female legislators by state
  ```sql
  ---Postgresql
  select
    first_state,gender,period,
    first_value(cohort_retained) over (partition by first_state,gender order by period) as cohort_size,
    cohort_retained,
    cohort_retained/first_value(cohort_retained) over (partition by first_state,gender order by period) as pct_retained
    from
    (select
    a.first_state,d.gender,
    coalesce(date_part('year',age(c.date,a.first_term)),0) as period,
    count(distinct a.id_bioguide) as cohort_retained
    from
    (select distinct id_bioguide,
    min(term_start) over (partition by id_bioguide order by term_start) as first_term,
    first_value(state) over (partition by id_bioguide order by term_start) as first_state
    from legislators_terms) a
    join legislators_terms b
    on a.id_bioguide=b.id_bioguide
    left join date_dim c
    on c.date between b.term_start and b.term_end
    and c.month_name='December' and c.day_of_month=31
    join legislators d 
    on a.id_bioguide=d.id_bioguide
    where a.first_term between '1917-01-01' and '1999-12-31'
    group by 1,2,3)aa
  ```
  * If visualising the output, we can see Alaska did not have any female legislators, while Arizona's female retention curve disappeared after year 3.
### Ensuring a record for every period so that the query returns zero values for retention instead of nulls.
* First Step
  * Querying for all combinations of periods and cohort attributes
  * In this case, *first_state* and *gender*, with the starting *cohort_size* for each combination
  * `CROSS JOIN` subquery aa(calculating the cohort) with a `genearate_series` subquery that returns all integers from 0 to 20
    ```sql
    ---Postgresql
    select
     aa.gender,aa.first_state,cc.period,aa.cohort_size
     from
     (select
     b.gender,
     a.first_state,
     count(distinct a.id_bioguide) as cohort_size
     from
     (select
     distinct id_bioguide,
     min(term_start) over (partition by id_bioguide) as first_term,
     first_value(state) over (partition by id_bioguide order by term_start) as first_state
     from legislators_terms)a
     join legislators b
     on a.id_bioguide=b.id_bioguide
     where a.first_term between '1917-01-01' and '1999-12-31'
     group by 1,2)aa
     cross join
     (
     select generate_series as period
     from generate_series(0,20,1))
     cc ;
    ```
* Next Step
  * `JOIN` this back to the actual periods in office, with a `LEFT JOIN` to ensure all time periods remain in the final result.
    ```sql
    ---Postgresql
    select aaa.gender,aaa.first_state,aaa.period,aaa.cohort_size,
          coalesce(ddd.cohort_retained,0) as cohort_retained,
          1.0*coalesce(ddd.cohort_retained,0)/aaa.cohort_size as pct_retained
          from
          (select
          aa.gender,aa.first_state,cc.period,aa.cohort_size
          from
          (select
          b.gender,
          a.first_state,
          count(distinct a.id_bioguide) as cohort_size
          from
          (select
          distinct id_bioguide,
          min(term_start) over (partition by id_bioguide) as first_term,
          first_value(state) over (partition by id_bioguide order by term_start) as first_state
          from legislators_terms)a
          join legislators b
          on a.id_bioguide=b.id_bioguide
          where a.first_term between '1917-01-01' and '1999-12-31'
          group by 1,2)aa
          cross join
          (
          select generate_series as period
          from generate_series(0,20,1))
          cc 
          )aaa
          left join
          (
          select
          d.first_state,
          g.gender,
          coalesce(date_part('year',age(f.date,d.first_term)),0) as period,
          count(distinct d.id_bioguide) as cohort_retained
          from
          (select distinct id_bioguide,
          min(term_start) over (partition by id_bioguide) as first_term,
          first_value(state) over (partition by id_bioguide order by term_start) as first_state
          from legislators_terms)d
          join legislators_terms e 
          on d.id_bioguide=e.id_bioguide
          left join date_dim f 
          on f.date between e.term_start and e.term_end
          and f.month_name='December' and f.day_of_month=31
          join legislators g on d.id_bioguide=g.id_bioguide
          where d.first_term between '1917-01-01' and '1999-12-31'
          group by 1,2,3)ddd
          on aaa.gender=ddd.gender and aaa.first_state=ddd.first_state
          and aaa.period=ddd.period;
    ```
## Defining Cohorts from Dates Other Than the First Date
