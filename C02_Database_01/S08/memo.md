# SQL - 집계와 그룹핑

## 집계와 그룹핑 실습 데이터 준비
- 테이블 생성
- 예제 데이터 삽입

---

## 집계 함수

### Aggregate Function
- AVG(expression): 표현식의 평균값을 반환
- COUNT(expression): 표현식의 결과가 NULL이 아닌 행의 수를 반환
- COUNT(*): 테이블 전체 행 수를 반환
- MAX(expression): 표현식의 최댓값을 반환
- MIN(expression): 표현식의 최솟값을 반환
- SUM(expression): 표현식의 합게를 반환

## 전체 데이터 건수 파악하기: COUNT()
- 특정 컬럼의 행 개수를 세는 집계함수
```sql
select count(*) from order_stat;

select count(category) from order_stat;
-- null을 빼고 게산함

select 
    count(*) as `전체 주문 건수`,
    count(category) as `카테고리 건수`
from 
    order_stat;
```

### 합게와 평균 게산: SUM(), AVG()
- SUM(컬럼명): 지정한 숫자 컬럼의 모든 값을 더하여 함계를 계산
- AVG(컬럼명): 지정한 숫자 컬럼의 값들의 평균을 계산
- **NULL**을 제외하고 계산함
```sql
select 
    sum(price * quantity) as `총 매출액`,
    avg(price * quantity) as `평균 주문 금액`
from
    order_stat;
```

### 최대, 최솟값: MAX(), MIN()
- MAX(): 지정한 컬럼에서 최댓값을 찾음
- MIN(): 지정한 컬럼에서 최솟값을 찾음
```sql
select 
    max(price) as `최고가`,
    min(price) as `최저가`
from 
    order_stat;
```

### 고유 고객 수 파악하기: DISTINCT
- 우리 쇼핑몰을 통해 주문한 고객
```sql
select count(distinct customer_name) from order_stat
```
- 풀어보자면
    - select distinct customer_name from order_stat
    - select count(distinct customer_name) from order_stat

---

## GROUP BY - 그룹으로 묶기

### 개념
- 특정 컬럼의 값이 같은 행동을 하나의 그룹으로 묶어주는 역할을 함
- 이것을 사용하면 전체에 대한 통계가 아닌 각 그룹에 대한 통계를 낼 수 있음

### 카테고리별 주문 건수
```sql
select 
    category,
    count(*) as `카테고리별 주문 건수`
from 
    order_stat
group by
    category;
```
- 카테고리별로 묶고, 각각을 기준으로 실행됨
- **null인 데이터도 하나의 그룹으로 취급함**

### 고객별로 몇 번 주문을 했을까?
```sql
select 
    customer_name,
    count(*) as `고객별 주문 횟수`
from 
    order_stat
group by
    customer_name;
```

### 그룹별로 심층 분석하기: GROUP BY와 집게함수
```sql
select 
    customer_name,
    count(*) as `고객별 주문 횟수`,
    sum(quantity) as `총 구매 수량`,
    sum(price * quantity) as `총 구매 금액`
from 
    order_stat
group by
    customer_name
order by
    `총 구매 금액` desc;
```
- 주의: order by 벡틱 사용
    - MySQL에서 작음 따옴표, 큰 따옴표를 사용하면 문자 상수로 인식함
    - 따라서 select에서 사용한 컬럼명으로 인식하지 못함
        - 문자 상수로 사용하면 모든 컬럼이 총 구매 금액이라는 문자 값으로 정렬되므로 정렬 효과가 없음

### 더 세분화된 그룹으로 분석하기: 여러 컬럼 기준으로 그룹화
- 어떤 고객이 어떤 카테고리의 상품을 주문하는지 보고 싶음
```sql
select 
    customer_name,
    category,
    sum(price * quantity) as `카테고리별 구매 금액`
from
    order_stat
group by
    customer_name, category
order by
    customer_name, `카테고리별 구매 금액` desc;
```

---

## GROUP BY - 주의사항
- select 절에는 group by에 사용된 컬럼과 집계 함수만 사용할 수 있음!!!
- 대전제
    - 그룹으로 묶고, 하나의 대푯값으로 뽑을 수 있어야 함
```sql
select 
    category,
    min(product_name),
    max(quantity),
    count(*),
    sum(quantity)
from
    order_stat
group by
    category;
```

---

## HAVING - 그룹 필터링 1

### Group By의 한계
```sql
select
    category,
    sum(price * quantity) as total_sales
from
    order_stat
where 
    sum(price * quantity) >= 500000
group by
    category;
```
- 먼저 각 카테고리별로 총 매출액을 계산
- where를 넣어서 매출액이 500000이 넘는 카테고리를 볼 수 있지 않을까?
    - NO!!
- 이유
    - where는 그룹화가 이뤄지기 전에 수행됨

### SQL 작동 순서
- from -> where -> group by -> ... -> select -> order by
- 따라서 그룹화를 하고 다시 필터링을 하고 싶다면
    - HAVING을 사용해야 함

### WHERE, HAVING 비유
- 전체 학생들이 운동자에 모여있음
- 안경 쓴 학생들만 앞으로 나와 (where)
- 앞으로 나온 학생들을 대상으로 같은 반 학생들끼리 모여 (group by)
- 마지막으로 이 중에서 반 평균 점수가 80점 이사인 그룹만 남아 (having)
```sql
select
    category,
    sum(price * quantity) as total_sales
from
    order_stat
group by
    category
having
    total_sales >= 500000;
```

### having 사용법
- having절의 조건문에서는 sum(), count(), avg() 같은 집게 함수를 사용할 수 있음
- group by를 통해서 그룹화하고 난 후 having으로 필터링
- 작동 순서
    - from where group by -> having -> select -> order by
- having에서 별칭을 사용할 수 있음
    - 근데 이건 MySQL에서는 동작하는데
    - 다른 RDB에서는 안됨
        - 이유는 having을 한 후 select를 사용해서, select의 별칭은 원칙적으로 사용을 못하지만
        - MySQL이 해줌
        - SQL 표준 스펙은 아님
        - 사용 안하는 것이 좋음

### 충성 고객 필터링하기
- 3회 이상 주문한 고객
```sql
select 
    customer_name,
    count(*) as order_count
from
    order_stat
group by
    customer_name
having
    count(*) >= 3;
```

---

## HAVING - 그룹 필터링 2

### WHERE vs HAVING
- 작동 시점
    - group by 이전
    - group by 이후
- 필터링 대상
    - 개별 행
    - 그룹
- 사용 가능 함수
    - 집계 함수 사용 불가
    - 집게 함수 사용 가능
- 역할
    - 개별 행을 선택
    - 그룹화된 결과 중에서 조건에 맞는 그룹만 선택

---

## SQL 실행 순서
- SELECT, FROM, WHERE, GROUP BY, HAVING, ORDER BY

### 논리적 실행 순서
- FROM: 가장 먼저 실행됨
- WHERE: FROM에서 가져온 테이블의 개별 행을 필터링함 / GROUP BY로 묶이기 전 날 것 그대로의 데이터를 1차로 걸러내는 단계
- GROUP BY: WHERE 절의 필터링을 통과한 행들을 기준으로 그룹을 형성함
- HAVING: GROUP BY로 통해 만들어진 그룹을 필터링함
- SELECT: 최종으로 조회
- ORDER BY: SELECT 절에서 선택된 최종 결과 후보들을 지정된 순서로 정렬
    - SELECT보다 늦게 수행되서 SELECT에서 사용한 별칭을 사용할 수 있음
- LIMIT: 정렬된 결과 중에서 최종적으로 사용자에게 반환할 행의 개수를 제한함

> 논리적으로는 이렇게 수행되지만, 성능 최적화를 위해 내부적으로 이 순서를 재배열해 물리적 실행 순서를 결정함. 물리적 순서가 바뀌더라도 엔진은 동일한 결과를 보장하므로, 논리적 실행 순서에 맞춰 쿼리를 작성하면 됨

### 예제
```sql
select
    customer_name,
    sum(price * quantity) as total_purchase
from
    order_stat
where
    order_date < '2025-05-14'
group by
    customer_name
having
    count(*) >= 2
order by
    total_purchase desc
limit 1;
```