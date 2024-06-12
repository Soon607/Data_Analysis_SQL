## Conversion Rate based on event_date, category_id
```sql
select
a.event_time,a.category_id,
round((1.0*a.active_user/b.total_user)*100,1) as conversion_rate
from
(select
event_time,
category_id,
count(*) as active_user
from data
where event_type='purchase'
group by 1,2
order by 1,2)a
join
(select
event_time,
category_id,
count(*) as total_user
from data
group by 1,2
order by 1,2)
b
on a.event_time=b.event_time and a.category_id=b.category_id
order by a.event_time,a.category_id
;
```
----
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
## Conversion Rate based on event_date
```sql
select
a.event_time,
100*(1.0*a.active_user/b.total_user) as conversion_rate
from
(select
event_time,
count(*) as active_user
from data
where event_type='purchase'
group by 1
order by 1)a
join
(select
event_time,
count(*) as total_user
from data
group by 1
order by 1)
b
on a.event_time=b.event_time;
```
## Monthly_Conversion_Rate
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
## Number of DailySessions per User
```sql
select
a.event_time,a.user,b.user_session,round(1.0*b.user_session/a.user,1) as sessions_per_user
from
(select
event_time,count(distinct user_id) as user
from data
group by event_time
order by event_time)a
join
(select
event_time,count(user_session) as user_session
from data
group by event_time
order by event_time)b
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
### Updating Table
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
