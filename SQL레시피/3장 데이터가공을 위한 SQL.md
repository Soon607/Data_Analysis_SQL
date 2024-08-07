# 5강 하나의 값 조작하기
* **데이터를 가공해야 하는 이유**
  * 다룰 데이터가 데이터 분석 용도로 상정되지 않은 경우
  * 연산할 때 비교 가능한 상태로 만들고 오류를 회피하기 위한 경우(데이터 형식을 일치시키기)
## 1. 코드 값을 레이블로 변경하기
- case 식
```sql
select
user_id,
case
when register_device=1 then 'desk top'
when register_device=2 then 'smart phone'
when register_device=3 then 'application'
else ''
end as device_name
from mst_users
```
## 2. URL에서 요소 추출하기
### <레퍼러로 어떤 웹 페이지를 거쳐 넘어왔는지 판별하기>
**레퍼러**: 어떤 웹 페이지를 거쳐 넘어왔는지를 판별
```sql
---페이지 단위로 집계하면 밀도가 너무 작아 복잡해지므로, 일반적으로 호스트 단위로 집계

select
stamp,
---호스트 이름 부분 추출하기
---substring 함수와 정규 표현식 사용하기
substring(referrer from 'https?://([^/]*)') as referrer_host
from access_log
```
****
* **'https?://([^/]*)'**
  * **' https? '**: 문자열이 ' http ' 또는 ' https '로 시작함을 의미('s?'는 's'가 있을 수도 없을수도 있다는 것을 의미)
  * **' :// '**: ' : '와 ' // ' 문자가 그 뒤에 오는 것을 의미
  * **' ([^/]*) '**: 첫 번째 ' / ' 문자가 나오기 전까지의 모든 문자를 캡처
    * **' [^/] '**: ' / '를 제외한 문자
    * **' * '**: 이전에 정의된 패턴이 0번 이상 반복될 수 있음을 의미
    * **' ( ) '**: 괄호 안에 있는 패턴을 캡처 그룹으로 만든다. 
  
### <URL에서 경로와 요청 매겨변수 값 추출하기>
```sql
stamp,url,
substring(url from '//[^/]+([^?#]+)') as path,
substring(url from 'id=([^&]*)') as id
from access_log
```
****
* **'//[^/]+([^?#]+)'**
  * **' // '**: 두 개의 슬래시로 시작함을 의미
  * **' [^/]+ '**: 슬래시가 아닌 문자가 하나 이상 연속됨을 의미
  * **' ([^?#]+) '**: 괄호 안의 부분은 캡처 그룹을 나타내며 이는 '?' 또는 '#'가 나오기 전까지의 모든 문자를 포함
* **'id=([^&]*)'**
  * **' id= '**: 문자열이 'id='로 시작함을 의미
  * **' ([^&]*) '**: 첫 번째 '&'문자가 나오기 전까지의 모든 문자를 캡처
    * **' * '**: 이전에 정의된 패턴이 0번 이상 반복될 수 있음을 의미
    * **'( )'**: 괄호 안에 있는 패턴을 캡처 그룹으로 만든다.
## 3. 문자열을 배열로 분해하기
* URL 경로를 슬래시로 분할해서 계층을 추출하는 쿼리
```sql
select
stamp,
url,
split_part(substring(url from '//[^/]+([^?#]+)'),'/',2) as path1,
split_part(substring(url from '//[^/]+([^?#]+)'),'/',3) as path2
from access_log
```
## 4. 날짜와 타임스탬프 다루기
### <현재 날짜와 타임스탬프 추출하기>
* 현재 날짜와 타임스탬프를 추출하는 쿼리
```
sql
select
current_date as dt,
--- 타임존이 적용된 타임스탬프
current_timestamp as stamp,
--- 타임존이 적용되지 않은 타임스탬프
localtimestamp as stamp_1;
```
### <지정한 값의 날짜/시각 데이터 추출하기>
* 문자열을 날짜 자료형, 타임스탬프 자료형으로 변환하는 쿼리
```sql
select
cast('2016-01-30' as date) as dt,
cast('2016-01-30 12:00:00 as timestamp) as stamp,

'2016-01-30'::date as date,
'2016-01-30 12:00:00'::timestamp as stamp
```
### <날짜/시각에서 특정 필드 추출하기>
* 타임스탬프 자료형의 데이터에서 연,월,일 등을 추출하는 쿼리
```sql
stamp,
extract(year from stamp) as year,
extract(month from stamp) as month,
extract(day from stamp) as day,
extract(hour from stamp) as hour
from
(select cast('2016-01-30 12:00:00' as timestamp) as stamp) as t;
```
* 타임스탬프를 나타내는 문자열에서 연,월,일 등을 추출하는 쿼리
```sql
select
stamp,
substring(stamp,1,4) as year,
substring(stamp,6,2) as month,
substring(stamp,9,2) as day,
substring(stamp,12,2) as hour,
substring(stamp,1,7) as year_month
from
(select cast('2016-01-30 12:00:00' as text) as stamp) as t
```
## 5. 결손값을 디폴트 값으로 대치하기
* 구매액에서 할인 쿠폰 값을 제외한 매출 금액을 구하는 쿼리
```sql
select
purchase_id,
amount,
coupon,
amount-coupon as discount_amount1,
amount-coalesce(coupon,0) as discount_amount2
from
purchase_log_with_coupon
```
# 6강 여러 개의 값에 대한 조작
## 1. 문자열 연결하기
* 문자열을 연결하는 쿼리
```sql
select
user_id,
concat(pref_name,city_name) as pref_city
from
mst_user_location
```
## 2. 여러 개의 값 비교하기
### <분기별 매출 증감 판정하기>
* q1,q2 컬럼을 비교하는 쿼리
```sql
select
year,
q1,q2,
case
when q1<q2 then '+'
when q1=q2 then ' '
else '-'
end as judge_q1_q2,
q1-q2 as diff_q2_q1,
sign(q2-q1) as sign_q2_q1
from
quarterly_sales
order by year
```
### <연간 최대/최소 4분기 매출 찾기>
* 연간 최대/최소 4분기 매출 찾기(greatest,least 함수 사용)
```sql
select
year,
greatest(q1,q2,q3,q4) as greatest_sales,
least(q1,q2,q3,q4) as least_sales
from quartely_sales
order by year
```
### <연간 평균 4분기 매출 계산하기>
* 단순한 연산으로 평균 4분기 매출 구하기
```sql
select
year,
(q1,q2,q3,q4)/4 as average
from quarterly_sales
order by year;
```
* coalesce를 사용해 null을 0으로 변환하고 평균값 구하기
```sql
select
year,
(coalesce(q1,0)+coalesce(q2,0)+coalesce(q3,0)+coalesce(q4,0))/4 as average
from
quarterly_sales
order by year
```
* q1/q2 매출만으로 평균을 구하기
```sql
select
year,
(coalesce(q1,0)+coalesce(q2,0)+coalesce(q3,0)+coalesce(q4,0))/
(sign(coalesce(q1,0))+sign(coalesce(q2,0))+sign(coalesce(q3,0))+sign(coalesce(q4,0))) as average
from quarterly_sales
order by year;
```
## 3. 2개의 값 비율 계산하기
**하나의 레코드에 포함된 값**을 조합하여 비율을 계산하기
### <정수 자료형의 데이터 나누기>
* 정수 자료형의 데이터를 나누는 쿼리
```sql
select
dt,ad_id,
cast(clicks as double precision)/impressions as ctr,
100.0*clicks/impressions as ctr_as_percent
from
advertising_stats
where
dt='2017-04-01'
order by dt,ad_id;
```
* 0으로 나누는 것을 피해 CTR을 계산하는 쿼리
```sql
select
dt,ad_id,
--- case식을 통해 분모가 0일 경우를 분기해서, 0으로 나누지 않게 하기
case
when impressions>0 then 100.0*clicks/impressions end as ctr_as_percent_by_case,
--- 분모가 0이라면 null로 변환해서, 0으로 나누지 않게 하는 방법
100.0*clicks/nullif(impressions,0) as ctr_as_percent_by_null
from advertising_stats
order by dt,ad_id
```
## 4. 두 값의 거리 계산하기
* 평균에서 어느 정도 떨어져있는지, 작년 매출과 올해 매출에 어느 정도 차이가 있는지를 확인 가능
### <숫자 데이터의 절댓값,제곱 평균 제곱근 계산하기>
* 1차원 데이터의 절댓값과 제곱 평균 제곱근을 계산하는 쿼리
```sql
select
abs(x1-x2) as abs,
sqrt(power(x1-x2,2)) as rms
from location_id
```
### <xy평면 위에 있는 두 점의 유클리드 거리 계산하기>
* 유클리드 거리 계산하기
```sql
select
sqrt(power(x1-x2,2)+power(y1-y2,2)) as dist,
point(x1,y1)<->point(x2,y2) as dist
from location_2d
```
## 5. 날짜/시간 계산하기
* 미래 또는 과거의 날짜/사간을 계산하는 쿼리
```sql
select
user_id,
register_stamp::timestamp as register_stamp,
register_stamp::timestamp+'1 hour'::interval as after_1_hour,
register_stamp::timestamp-'30 minutes'::interval as before_30_minutes,
register_stamp::date as register_date,
(register_stamp::date+'1 day'::interval)::date as after_1_day,
(register_stamp::date-'1 month'::interval)::date as before_1_month
from mst_users_with_dates
```
### <날짜 데이터들의 차이 계산하기>
* 두 날짜의 차이를 계산하는 쿼리
```sql
select
user_id,
current_date as today,
register_stamp::date as register_date,
current_date-register_stamp::date as diff_days
from mst_users_with_dates;
```
### 사용자의 생년월일로 나이 계산하기
* age함수를 사용해 나이를 계산하는 쿼리
```sql
select
user_id,
current_date as today,
register_stamp::date as register_date,
birth_date::date as birth_date,
extract(year from age(birth_date::date)) as current_age,
extract(year from age(register_stamp::date,birth_date::date)) as register_age
from mst_users_with_dates
```
* 등록 시점과 현재 시점의 나이를 문자열로 계산하는 쿼리
```sql
select
user_id,
substring(register_stamp,1,10) as register_date,
birth_date,
---등록 시점의 나이 계산하기
floor(
(cast(replace(substring(register_stamp,1,10),'-','')as integer)
-cast(replace(birth_date,'-','')as integer))/10000) as register_age,
floor(
cast(replace(cast(current_date as text),'-','')as integer)
-cast(replace(birth_date,'-','')as integer))/1000)
as current_age
from mst_users_with_dates
```
## 6.IP주소 다루기
### <IP 주소 자료형 활용하기>
* **inet** 자료형을 사용한 IP주소 비교 쿼리
```sql
select
cast('127.0.0.1' as inet)<cast('127.0.0.2' as inet) as lt,
cast('127.0.0.1' as inet)>cast('192.168.0.1' as inet) as gt;
```
* address/y 형식의 네트워크 범위에 IP주소가 포함되는지 판정하기
```sql
select
cast('127.0.0.1' as inet)<<cast('127.0.0.0/8' as inet) as is_contained
```
