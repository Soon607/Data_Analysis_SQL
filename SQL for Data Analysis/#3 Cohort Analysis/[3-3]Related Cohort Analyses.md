# Related Cohort Analysis
Answering to questions like
* how long an entity lasted,
* whether an entity did multiple actions and how many of those actions occurred. 
## Survivorship(survival analysis)
* Considering *how long something lasts*, *the duration of time until a particular event such as churn or death
* Answering questions about the share of the population that is likely to remain past a certain amount of time
* Cohorts can help identify or at least provide hypotheses about which characteristics or circumstances increase or decrease the survival likelihood

### Survivorship Calculation
* Similar to a *Retention Analysis*, but instead of calculating whether an entity was present in a certain period, it calculates whether the entity is present in that period or later in the time series.
* After that, we calculated the share of the total cohort
* Typically, one or more periods are chosen depending on the nature of the data set analysed.
### Looking at the share of legislators who survived in office for a decade or more after their first term started
* First, calculating the first and last *term_start* dates(don't need to know the specific dates of each term)
  ```sql
  ---Postgresql
  select
      id_bioguide,
      min(term_start) as first_term,
      max(term_start) as last_term
      from legislators_terms
      group by 1;
  ```
* Finding century of the *min_term_start*, and calculating the *tenure* as the number of years between the *min* and *max* *term_starts*
  ```sql
  ---Postgresql
  select
    id_bioguide,
    date_part('century',min(term_start)) as first_century,
    min(term_start) as first_term,
    max(term_start) as last_term,
    date_part('year',age(max(term_start),min(term_start))) as tenure
    from legislators_terms
    group by 1;
  ```
* Last, Calculating the *cohort_size* with a count of all the legislators and the number who survived for at least 10 years
  ```sql
  ---Postgresql
  select
    first_century,
    count(distinct id_bioguide) as cohort_size,
    count(distinct case when tenure>=10 then id_bioguide end) as survived_10,
    1.0*count(distinct case when tenure>=10 then id_bioguide end)/count(distinct id_bioguide) as pct_survived_10
    from
    (select
    id_bioguide,
    date_part('century',min(term_start)) as first_century,
    min(term_start) as first_term,
    max(term_start) as last_term,
    date_part('year',age(max(term_start),min(term_start))) as tenure
    from legislators_terms
    group by 1)a
    group by 1
  ```
*****
* Calculating the share of legislators in each century who survived for five or more total terms, in case terms may or may not be consecutive.
  ```sql
  ---Postgresql
  select
    first_century,
    count(distinct id_bioguide) as cohort_size,
    count(distinct case when total_terms>=5 then id_bioguide end) as survived_5,
    1.0*count(distinct case when total_terms>=5 then id_bioguide end)/
    count(distinct id_bioguide) as pct_survived_5_terms
    from
    (select
    id_bioguide,
    date_part('century',min(term_start)) as first_century,
    count(term_start) as total_terms
    from legislators_terms
    group by 1)a
    group by 1
  ```
* Calculating the survivorship for each number of years or periods
  ```sql
  ---Postgresql
  select
      a.first_century,b.terms,
      count(distinct id_bioguide) as cohort,
      count(distinct case when a.total_terms>=b.terms then id_bioguide end) as cohort_survived,
      1.0*count(distinct case when a.total_terms>=b.terms then id_bioguide end)/count(distinct id_bioguide)
      as pct_survived
      from
      (select
      id_bioguide,
      date_part('century',min(term_start)) as first_century,
      count(term_start) as total_terms
      from legislators_terms
      group by 1)a
      cross join
      (select generate_series as terms
      from generate_series(1,20,1))
      b 
      group by 1,2
  ```
  * Calculated the survivorship for each number of terms from 1 to 20. (Using `generate_series`)
  * The calculation was accomplished by using a `cross-join.`

### Summary
* Survivorship is closely related to retention.
* While retention counts entities present in a specific number of periods from the start, survivorship considers only *whether an entity was present as of a specific period or later.
* It needs only *the first* and *last dates* in the time series to calculate

## Returnship or Repeat Purchase Behavior
* Understanding whether a cohort member can be expected to return within a given window of time and the intensity of activity during that window
* For example, an e-commerce site might want to know not only how many new buyers were acquired but also whether those buyers have become *repeat buyers*
* The analysis must be conducted with the same lifespan.
* To make fair comparisons between cohorts with different starting dates
  * We need to create an analysis based on a *time box*, or a fixed window of time from first date
  * Therefore, every cohort has an equal amount of time under consideration
### Example(Legislators data)
* How many legislators have more than one term type, and specifically, what share of them start as representatives and go on to become senators
* First, finding the cohort size for each century, using the `subquery` and `date_part`
  ```sql
  ---Postgresql
  select
    date_part('century',first_term) as cohort_century,
    count(id_bioguide) as reps
    from
    (select
    id_bioguide,min(term_start) as first_term
    from legislators_terms
    where term_type='rep'
    group by 1)a
    group by 1
  ```
* Finding the representative who later became senators
  ```sql
  ---Postgresql
  select
      date_part('century',a.first_term) as cohort_century,
      count(distinct a.id_bioguide) as rep_and_sen
      from
      (select id_bioguide, min(term_start) as first_term
      from legislators_terms
      where term_type='rep'
      group by 1)a
      join legislators_terms b 
      on a.id_bioguide=b.id_bioguide and
      b.term_type='sen' and b.term_start>=a.first_term
      group by 1
  ```
  * Used a `JOIN` to the *legislators_terms* table
  * Used a clauses *b.term_type='sen'* and *b.term_start>a.first_term*
* Calculating the percentage of representatives who became senators
  ```sql
  ---Postgresql
  select aa.cohort_century,
      1.0*bb.rep_and_sen/aa.reps as pct_rep_and_sen
      from
      (select
      date_part('century',first_term) as cohort_century,
      count(id_bioguide) as reps
      from
      (select id_bioguide,min(term_start) as first_term
      from legislators_terms
      where term_type='rep'
      group by 1)a
      group by 1)aa
      left join
      (select
      date_part('century',b.first_term) as cohort_century,
      count(distinct b.id_bioguide) as rep_and_sen
      from
      (select id_bioguide,min(term_start) as first_term
      from legislators_terms
      where term_type='rep'
      group by 1)b
      join legislators_terms c 
      on b.id_bioguide=c.id_bioguide and
      c.term_type='sen' and c.term_start>b.first_term
      group by 1)bb
      on aa.cohort_century=bb.cohort_century
  ```
  * A `LEFT JOIN` was used
    * Ensuring that all cohorts are included whether or not the subsequent event happened.
* The above result has yet to contain a **time box** to ensure a fair comparison.
* Assuming that all legislators who served in the 18th and 19th centuries are no longer living, many of those who were first elected in the 20th and 21st centuries are still in the middle of their careers
  * Adding the filter `WHERE age(c.term_start,b.first_term)<=interval '10 years'` to *subquery bb* creates a time box of 10
  * In addition, adding `WHERE first_term<='2009-12-31'` to *subquery a* to exclude those who were less than 10 years into their careers when the data set was assembled
    ```sql
    ---Postgresql
    select aa.cohort_century,
          1.0*bb.rep_and_sen/aa.reps as pct_10_yrs
          from
          (select
          date_part('century',first_term) as cohort_century,
          count(id_bioguide) as reps
          from
          (select id_bioguide,min(term_start) as first_term
          from legislators_terms
          where term_type='rep'
          group by 1)a
          where first_term<='2009-12-31'
          group by 1)aa
          left join
          (select
          date_part('century',b.first_term) as cohort_century,
          count(distinct b.id_bioguide) as rep_and_sen
          from
          (select id_bioguide,min(term_start) as first_term
          from legislators_terms
          where term_type='rep'
          group by 1)b
          join legislators_terms c 
          on b.id_bioguide=c.id_bioguide and
          c.term_type='sen' and c.term_start>b.first_term
          where age(c.term_start,b.first_term)<=interval '10 years'
          group by 1)bb
          on aa.cohort_century=bb.cohort_century
    ```
    * The 18th century had the highest share of representatives becoming senators within ten years
    * 21st: The second-highest
* **Comparing several time windows**
  * First Option: Running the query several times with different intervals and windows
  * Second Option: Calculating multiple windows in the same result set by using a set of `CASE` statements inside of *count* `distinct` aggregations to form the intervals, rather than specifying the interval in the `WHERE` clause
    ```sql
    ---Postgresql
    select
      aa.cohort_century,
      bb.rep_and_sen_5_yrs*1.0/aa.reps as pct_5_yrs,
      bb.rep_and_sen_10_yrs*1.0/aa.reps as pct_10_yrs,
      bb.rep_and_sen_15_yrs*1.0/aa.reps as pct_15_yrs
      from
      (select
      date_part('century',first_term) as cohort_century,
      count(distinct id_bioguide) as reps
      from
      (select id_bioguide,min(term_start) as first_term
      from legislators_terms
      where term_type='rep'
      group by 1)a
      where first_term<='2009-12-31'
      group by 1)aa
      left join
      (select
      date_part('century',b.first_term) as cohort_century,
      count(distinct case when age(c.term_start,b.first_term)<=interval '5 years' then b.id_bioguide end) as rep_and_sen_5_yrs,
      count(distinct case when age(c.term_start,b.first_term)<=interval '10 years' then b.id_bioguide end) as rep_and_sen_10_yrs,
      count(distinct case when age(c.term_start,b.first_term)<=interval '15 years' then b.id_bioguide end) as rep_and_sen_15_yrs
      from
      (select
      id_bioguide, min(term_start) as first_term
      from legislators_terms
      where term_type='rep'
      group by 1)b
      join
      legislators_terms c 
      on b.id_bioguide=c.id_bioguide
      and c.term_type='sen' and c.term_start>b.first_term
      group by 1)bb
      on aa.cohort_century=bb.cohort_century
    ```
    * Can see how the share of representatives who became senators evolved over time, both within each cohort and across cohorts
   
### Summary
* Finding the repeat behaviour within a fixed time box is a useful tool for comparing cohorts.
* When the behaviours are intermittent in nature, such as purchase behaviour or content or service consumption.

## Cumulative Calculations
* For establishing *cumulative lifetime value*(called *customer lifetime value*) and monitoring newer cohorts in order to predict what their full LTV will be.
* It is possible because early behaviour is often highly correlated with long-term behaviour
  * Users of a service who return frequently in their first days or weeks of using it tend to be the most likely to stay around over the long term.
  * Customers who buy a second or third time early on are likely to continue purchasing over a longer time period.
  * Subscribers who renew after the first month or year are often likely to stick around over many subsequent months or years
* This analysis can also be applied to situations where customers or entities incur costs, such as through product returns, support interactions, or use of healthcare services.
* The analysis is less concerned about whether an entity did an action on a particular date and more about the total as of a particular date
### Example(Legislators data)
* Looking at the number of terms started within ten years of the *first_term_start*, cohorting the legislators by century and type of first term.
  ```sql
  ---Postgresql
  select
      date_part('century',a.first_term) as century,
      first_type,
      count(distinct a.id_bioguide) as cohort,
      count(b.term_start) as terms
      from
      (select
      distinct id_bioguide,
      first_value(term_type) over (partition by id_bioguide order by term_start) as first_type,
      min(term_start) over (partition by id_bioguide) as first_term,
      min(term_start) over (partition by id_bioguide) + interval '10 years' as first_plus_10
      from legislators_terms)a
      left join
      legislators_terms b 
      on a.id_bioguide=b.id_bioguide
      and b.term_start between a.first_term and a.first_plus_10
      group by 1,2
  ```
  * The result can tell the overall contribution of a cohort to an organisation.
* Total sales or total repeat purchases can be valuable metrics
* We want to normalize to understand the contribution on a *per-entity* basis
  * average actions per person, average order value, items per order, and orders per customer.
* To normalise the cohort size, simply divide by the starting cohort.
  ```sql
  ---Postgresql
  select
      century,
      max(case when first_type='rep' then cohort end) as rep_cohort,
      max(case when first_type='rep' then terms_per_leg end) as avg_rep_terms,
      max(case when first_type='sen' then cohort end) as sen_cohort,
      max(case when first_type='sen' then terms_per_leg end) as avg_sen_terms
      from
      (select
      date_part('century',a.first_term) as century,
      first_type,
      count(distinct a.id_bioguide) as cohort,
      count(b.term_start) as terms,
      1.0*count(b.term_start)/count(distinct a.id_bioguide) as terms_per_leg
      from
      (select
      distinct id_bioguide,
      first_value(term_type) over (partition by id_bioguide order by term_start) as first_type,
      min(term_start) over (partition by id_bioguide) as first_term,
      min(term_start) over (partition by id_bioguide) + interval '10 years' as first_plus_10
      from legislators_terms)a
      left join
      legislators_terms b 
      on a.id_bioguide=b.id_bioguide
      and b.term_start between a.first_term and a.first_plus_10
      group by 1,2)aa
      group by 1
  ```
### Summary
* Cumulative calculations are often used in customer lifetime value calculations
* LTV is usually calculated using monetary measures, such as total dollars spent by a customer or the gross margin generated by a customer across their lifetime
* To facilitate comparisons between cohorts, the **lifetime** is often chosen to reflect average customer lifetime, ore periods that are convenient to analyse, such as 3,5 or 10 years.
