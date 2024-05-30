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
## Conversion Rate base on event_date
``sql
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
