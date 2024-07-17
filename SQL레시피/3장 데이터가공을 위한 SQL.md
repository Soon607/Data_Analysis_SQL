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
