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
