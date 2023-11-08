# Cohorts: A useful Analysis Framework
* **A cohort**
  * A group of individuals who share some characteristic of interest
  * can be companies, products, or physical world phenomena
* **Cohort Analysis**
  * A useful way to compare groups of entities over time
  * Understanding changes in consumer behaviours weekly, monthly or yearly
  * Providing a framework for detecting correlations between cohort characters and these long-term trends.
    * can lead to hypotheses about the casual drivers.
  * *Understanding customers and generating hypotheses*
* **Three components in Cohort Analyses**
  * Grouping
    * can be done based on various elements
      * start date(first purchase, subscription)
      * Innate qualities(birth year, country of origin...)
      * Characteristics(city of residence, marital status)
  * Time Series
    * A series of purchases, logins, interactions, or other actions that are taken by the customers or entities.
    * Covering the entire lifespan of the entities
    * There will be a *Survivorship Bias*
      * Occurring when only customers who have stayed are in the data set as churned customers are excluded because they are no longer around, so the rest of the customers appear to be of higher quality or fit in comparison to newer cohorts
    * In order to normalise, cohort analysis usually measures the number of periods that have elapsed from a starting date rather than calendar months.
      * cohorts can be compared in period 1, period 2
      * The intervals may be days, weeks, months, or years.
  * Aggregate metric
    * related to the actions that matter to the health of the organisation
    * with `sum`, `count`, or `average`
    * The result is a time series that can then be used to understand changes in behaviour over time
* **4 types of cohort analysis**
  * Retention
    * observing whether the cohort member has a record in the time series on a particular date, expressed as a number of periods from the starting date
    * useful for organisations where repeated actions are expected.
    * help understand how sticky or engaging a product is and how many entities can be expected to appear on future dates.
  * Survivorship
    * observing how many entities remained in the data set for a certain length of time or longer, regardless of the number of or frequency of actions up to that time
    * help understand the proportion of the population that can be expected to remain either way
      * In a positive sense: by not churning or passing away
      * In a negative sense: by not graduating or fulfilling some requirement.
  * Returnship(Repeat purchase behaviour)
    * observing whether an action has happened more than some minimum threshold of times(simply more than once) during a fixed window of time
    * useful for organisations where the behaviour is intermittent and unpredictable.
  * Cumulative
    * concerned with the total number or amounts measured at one or more fixed time windows, regardless of when they happened during that window
    * are often used in calculations of customer lifetime value (LTV or CLTV)
