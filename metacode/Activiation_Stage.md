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
### Conver rate with number of session
```sql
select
a.event_time,
round(100*(1.0*a.active_user/b.total_user),1) as conversion_rate,
b.total_user,
a.active_user
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

## Final Table(Oct-Dec)
```sql
select a.event_time,a.total_user,a.active_user,concat(a.conversion_rate,'%') as conversion_rate,b.number_of_user_session,
concat('$',round(c.revenue,0)) as revenue from
daily_conversion_rate a
join daily_usersession b
on a.event_time=b.event_time
join daily_revenue c
on b.event_time=c.event_time
```
