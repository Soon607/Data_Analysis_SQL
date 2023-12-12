# The Data Sets
## game_users table
![image](https://github.com/Soon607/Data_Analysis_SQL/assets/98214539/a6986f85-cac8-4f09-bc8c-ac1fc5c84305)
## game_actions table
![스크린샷 2023-12-12 164653](https://github.com/Soon607/Data_Analysis_SQL/assets/98214539/639a45bd-fbe7-448b-9e67-08e326050d4a)
## game_purchases table
![image](https://github.com/Soon607/Data_Analysis_SQL/assets/98214539/8973cf86-f171-4497-9a87-f837de4e2520)
## exp_assignment table
* Contains records of which variant users were assigned to for a particular experiment
![image](https://github.com/Soon607/Data_Analysis_SQL/assets/98214539/c91885bf-46b4-4f07-925c-3e309b944c9f)
# Types of Experiments
There are two main types of experiments(Binary outcomes and Continous outcomes)
## Experiments with Binary Outcomes(The Chi-Squared Test)
* Has only two outcomes: either an action is taken, or it isn't
  * Whether a user completes a registration flow or they don't
  * Whether consumers click on a website ad or they don't
* Calculating the proportion of each variant that completes the action
  * The numerator: The number of completers
  * The denominator: All units that were exposed
* This metric is also described as a rate: completion rate, click-through rate, graduation rate, and so on.

Using the **chi-squared test** to determine whether the rates in the variants are statistically different.    
Data for a chi-squared test is often shown in the form of a *contingency table*, which shows the frequency of observations at the intersection of two attributes.
****
* Looking at whether a new onboarding flow increased the completion rate.
* Creating the contingency table that shows the frequency at the intersection of the variant assignment(control or variant 1) and whether of not onboarding was completed.
  
  ```sql
  ---Postgresql
  select
    a.variant,
    count(case when b.user_id is not null then a.user_id end) as completed,
    count(case when b.user_id is null then a.user_id end) as not_completed
    from 
    exp_assignment a
    left join game_actions b
    on a.user_id=b.user_id
    and b.action='onboarding complete'
    where a.exp_name='Onboarding'
    group by 1
  ```
To make use of one of the online significance calculators, we will need the number of successes, or times when the action was taken, and the total number cohort for each variant.
* Finding the percent completed in each variant
  ```sql
  ---Postgresql
  select
    a.variant,
    count(a.user_id) as total_cohorted,
    count(b.user_id) as completions,
    1.0*count(b.user_id)/count(a.user_id) as pct_completed
    from 
    exp_assignment a
    left join game_actions b
    on a.user_id=b.user_id
    and b.action='onboarding complete'
    where a.exp_name='Onboarding'
    group by 1
  ```
****
We can see that variant 1 did indeed have more completions than the control experience, with 76.14% completing compared to 72.69%.    
However, we cannot say there is a statistically significant difference that allows us to reject the null hypothesis.    
Therefore, we need to plug our results into an online calculator and confirm that the completion rate for variant 1 was significantly higher at a 95% confidence level than the completion for the control. (Variant 1 can be declared the winner.)
## Experiments with Continuous Outcomes: The t-Test
* Many experiments seek to improve *continuous metrics*, rather than the binary outcomes    
* Continuous metrics can take on a range of values
  * amount spent by customers, time spent on page, and days an app is used
 
E-commerce sites often want to increase sales, so they might experiment with product pages or checkout flows.    
Content sites may test layout, navigation, and headlines to try to increase the number of stories read.    
A company running an app might run a remarketing campaign to remind users to come back to the app.
****
* **The goal is to figure out whether the average values in each variant differ from each other in a statistically significant way**
* The relevant statistical test is the **two-sample t-test**
* In this section, we will look at whether that new flow increased user spending on in-game currency.
  * The success metric is the amount spent, so we need to calculate the mean and standard deviation of this value for each variant
* First, we need to calculate the amount per user since users can make multiple purchases.
  ```sql
  ---Postgresql
  select
    variant,
    count(user_id) as total_cohorted,
    avg(amount) as mean_amount,
    stddev(amount) as stddev_amount
    from
    (select a.variant,a.user_id,sum(coalesce(b.amount,0)) as amount
    from exp_assignment a
    left join game_purchases b
    on a.user_id=b.user_id
    where a.exp_name='Onboarding'
    group by 1,2)a
    group by 1
  ```
  * Next, we plug these values into an online calculator and find that there is no significant differences between the control and variant groups at a 95% confidence interval.
****
* Another question: Considering whether variant 1 affected spending among those users who completed the onboarding
  * Those who don't complete the onboarding never make it into the game and therefore, don't even have the opportunity to make a purchase
    ```sql
    select
      variant,
      count(user_id) as total_cohorted,
      avg(amount) as mean_amount,
      stddev(amount) as stddev_amount
      from
      (select
      a.variant,
      a.user_id,
      sum(coalesce(b.amount,0)) as amount
      from exp_assignment a
      left join game_purchases b
      on a.user_id=b.user_id
      join game_actions c
      on a.user_id=c.user_id
      and c.action='onboarding complete'
      where a.exp_name='Onboarding'
      group by 1,2)a
      group by 1
    ```
      
