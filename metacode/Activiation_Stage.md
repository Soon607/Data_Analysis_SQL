# Acquisition&Activiation Level
## User-Sessions per User
### Number of DailySessions per User
```sql
with data1 as(select
a.event_time,a.user,b.user_session,round(1.0*b.user_session/a.user,1) as sessions_per_user
from
(select
event_time,count(distinct user_id) as user
from data_20_22
group by event_time
order by event_time)a
join
(select
event_time,count(user_session) as user_session
from data_20_22
group by event_time
order by event_time)b
on a.event_time=b.event_time
order by a.event_time),
data2 as(
select event_time,round(sum(price),0) as revenue
from data_20_22
group by event_time
order by event_time)

select
a.event_time,a.user,a.user_session,a.sessions_per_user,b.revenue
from data1 a
join data2 b
on a.event_time=b.event_time
order by a.event_time
```
### Number of DailySessions per User(without remove_from_cart)
```sql
select
a.event_time,a.user,b.user_session,round(1.0*b.user_session/a.user,1) as sessions_per_user
from
(select
event_time,count(distinct user_id) as user
from data
where not event_type='remove_from_cart'
group by event_time
order by event_time)a
join
(select
event_time,count(user_session) as user_session
from data
where not event_type='remove_from_cart'
group by event_time
order by event_time)b
on a.event_time=b.event_time
order by a.event_time
```
****
## Conversion Rate
### Daily Conversion Rate
```sql
with data1 as(select
a.event_time,b.total_user,a.active_user,
round(100*(1.0*a.active_user/b.total_user),1) as conversion_rate
from
(select
event_time,
count(distinct user_id) as active_user
from data
where event_type='purchase'
group by 1
order by 1)a
join
(select
event_time,
count(distinct user_id) as total_user
from data
group by 1
order by 1)
b
on a.event_time=b.event_time)

select
a.event_time,a.total_user,a.active_user,concat(a.conversion_rate,'%') as conversion_rate,b.revenue
from data1 a
join daily_revenue b
on a.event_time=b.event_time
order by a.event_time;
```
*****
```sql
with data1 as(select
a.event_time,b.total_user,a.active_user,
round(100*(1.0*a.active_user/b.total_user),1) as conversion_rate
from
(select
event_time,
count(distinct user_id) as active_user
from data_101112
where event_type='purchase'
group by 1
order by 1)a
join
(select
event_time,
count(distinct user_id) as total_user
from data_101112
group by 1
order by 1)
b
on a.event_time=b.event_time)
,
data2 as(select event_time,sum(price) as revenue
from data_101112
where event_type='purchase'
group by event_time
order by event_time)

select
a.event_time,a.total_user,a.active_user,concat(a.conversion_rate,'%') as conversion_rate,b.revenue
from data1 a
join data2 b
on a.event_time=b.event_time
order by a.event_time;
```
### Monthly_Conversion_Rate
```sql
with 
monthly_purchase as(
select
event_time,
extract(year from event_time) as year,
extract(month from event_time) as month,
event_type,
user_session
from data
order by event_time)

select
concat(year,'-',month)  as event_time,
total_user,active_user,monthly_conversion_rate
from
(select
a.year,a.month,a.total_user,b.active_user,round(100*(1.0*b.active_user/a.total_user),2) as monthly_conversion_rate
from
(select year,month,count(*) as total_user
from monthly_purchase
group by 1,2
order by 1,2) a
join
(select year,month,count(*) as active_user
from monthly_purchase
where event_type='purchase'
group by 1,2
order by 1,2)b
on a.year=b.year and a.month=b.month
order by a.year,a.month)as c;
```
### Daily Conversion Rate(Brand 1-9)
```sql
with data1 as
(select event_time,brand,count(*) as quantity 
from new_data
where brand in (select brand from brand_top10)
group by event_time,brand
order by event_time),
data2 as(
select event_time,brand,count(*) as sold,sum(price) as revenue
from new_data
where event_type='purchase'
and brand in (select brand from brand_top10)
group by event_time,brand
order by event_time)

select
a.event_time, a.brand,a.quantity,b.sold,concat(round(1.0*b.sold/a.quantity,2),'%') as conversion_rate,round(b.revenue,0) as revenue
from data1 a
join data2 b
on a.event_time=b.event_time and a.brand=b.brand
join brand_top10c
on b.brand=c.brand
order by a.event_time,c.rank
```
### Daily Conversion Rate(Other Brands)
```sql
with data1 as
(select event_time,brand,count(*) as quantity 
from new_data
where brand not in (select brand from brand_ranking where not brand='other brands')
group by event_time,brand
order by event_time),
data2 as(
select event_time,brand,count(*) as sold,sum(price) as revenue
from new_data
where event_type='purchase'
and brand not in (select brand from brand_ranking where not brand='other brands')
group by event_time,brand
order by event_time)

select
event_time,sum(quantity) as quantity, sum(sold) as sold,concat(sum(round(1.0*sold/quantity,2)),'%') as conversion_rate,sum(revenue) as revenue
from
(select
a.event_time, a.brand,a.quantity,b.sold,concat(round(1.0*b.sold/a.quantity,2),'%') as conversion_rate,round(b.revenue,0) as revenue
from data1 a
join data2 b
on a.event_time=b.event_time and a.brand=b.brand
order by a.event_time,a.brand)a
group by event_time
order by event_time
```
# Other Querys
## Updating Table
```sql
drop table if exists new_data;

create table new_data(
id varchar(20),
event_time date,
event_type varchar(20),
category_id varchar(50),
brand varchar(20),
price numeric,
user_id varchar(20),
user_session varchar(50))
```
## Brand Ranking
```sql
with data as(select extract(month from event_time) as month,brand,round(sum(price),0) as revenue
from new_data
where event_type='purchase'
group by extract(month from event_time),brand
order by extract(month from event_time),revenue desc),
rank_data as
(SELECT month, brand, revenue, rank
FROM (
    SELECT month, brand, revenue,
           ROW_NUMBER() OVER (PARTITION BY month ORDER BY revenue DESC) AS rank
    FROM data
    WHERE month IN (10, 11, 12) AND brand IS NOT NULL
) AS ranked_data
ORDER BY month, rank)


SELECT month, 
       MAX(brand) AS brand,
       SUM(revenue) AS revenue
FROM rank_data
WHERE rank > 9
GROUP BY month
order by month
```
## Brand Ranking(2)
```sql
with data1 as(select
brand,revenue,row_number() over (order by revenue desc) as rank
from
(select brand,round(sum(price),0) as revenue
from new_data
where event_type='purchase' and brand is not null and event_time between '2019-10-1' and '2019-12-31'
group by 1
order by revenue desc)a)

select * from data1
where rank<10
```
## Adding Day Name Column
```sql
SELECT 
    event_time, 
    event_type, 
    category_id, 
    brand, 
    price, 
    user_id, 
    user_session,
    TO_CHAR(event_time, 'FMDay') AS day_name
FROM 
    new_data;
```
