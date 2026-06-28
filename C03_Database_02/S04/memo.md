# S04. 서브쿼리

## 서브쿼리 소개
- 조인을 통해 흩어진 테이블을 연결하는 법을 배움
- 조인만으로 한 번에 답하기 어려운, 여러 단계의 사고를 거쳐야하는 문제들이 있음

### 문제 상황
- 우리 쇼핑몰에서 판매하는 상품들의 평균 가격보다 비싼 상품은 무엇이 있을까?
```sql
select avg(price) from products; -- 평균 가격 구하기: 167166.6667

select name, price
from products
where price > 167166.6667;
```
- 이 과정을 하나로 합칠 수 없을까?
- 현재 문제점
    - 번거로움
    - 오류에 취약: 상품 데이터가 실시간으로 추가되거가 가격이 변경될 수 있음

- 해결: **서브쿼리**

### 서브쿼리의 개념
- 하나의 SQL 쿼리 문 안에 포함된 또 다른 SELECT 쿼리를 의미함
- 바깥쪽 메인 쿼리가 실행되기 전에, 괄호 안에 있는 서브쿼리가 먼저 실행됨
```sql
select name, price
from products
where price > (select avg(price) from products);
```

### 실행 순서
- 괄호 안의 서브쿼리를 먼저 실행
- 실행된 결과인 단일 값을 반환
- 내부적으로 단일 값과 비교
- 최종적으로 메인 쿼리 실행

### 종류와 특징
- where절 외에도 쿼리의 다양한 위치에서 활약 가능, 다른 역할을 수행
- 행과 컬럼의 수에 따라 종류가 나뉘며,
- 사용되는 위치와 연산자에 따라 그 역할이 결정됨

---

## 스칼라 서브쿼리
- 단일 컬럼, 단일 행 서브쿼리

### 스칼라
- 단, 하나의 값을 의미
- 단일 행 비교 연산자들(=, >, <, >=, <=, <>>)과 함께 사용할 수 있음

### 문제 상황
- 특정 주문(order_id = 1)을 한 고객과 같은 도시에 사는 모든 고객을 찾고 싶음
```sql
select 
	name, address
from 
	users
where 
	address = (select u.address
	           from orders o
               join users u on o.user_id = u.user_id
	           where o.order_id = 1);
```

### 스칼라 서브쿼리의 치명적 오류
- 반드시, 무슨 일이 있어도, 단 하나의 행만 반환해야함
- order_id, user_id처럼 PK나 UNIQUE 제약 조건이 걸린 컬럼을 조건으로 조회하는 경우가 대표적

### 그렇다면
- 전자기기 카테고리에 속하는 모든 상품들을 주문한 고객 목록을 찾아라
- 이때는 = 같은 단일행 연산자가 아닌, 여러 개의 값을 다룰 수 있는 IN과 같은 다중 행 연산자를 사용해야함

---

## 다중 행 서브쿼리

### 문제 상황
- 전자기기 카테고리에 속한 모든 상품들을 주문한 주문 내역을 전부 보고 싶음
	- 먼저, 전자기기 카테고리에 속한 상품들의 product_id를 모두 찾음
	- 그다음, orders 테이블에서 product_id가 우리가 찾아낸 product_id 목록 안에 포함된 주문들을 모두 찾음

### IN
- where 컬럼명 in (값1, 값2, ...)
```sql
select * from orders
where product_id in (select product_id from products where category = '전자기기')
order by order_id;
```

### ANY, ALL 연산자: 목록의 모든/일부 값과 비교

- .> ANY (서브쿼리): 서브쿼리가 반환한 여러 결과값 중 어느 하나보다만 크면 참. 즉, 최솟값보다 크면 참
- .> ALL (서브쿼리): 서브쿼리가 반환한 여러 결과값 모두보다 커야만 참. 즉, 최대값보다 커야 참
- < ANY (서브쿼리): 최대값보다 작으면 참
- < ALL (서브쿼리): 최소값보다 작으면 참
- = ANY (서브쿼리): IN과 완전히 동일한 의미, 목록 중 어느 하나와 같으면 참

### 문제 상황 1
- 전자기기 카테고리의 어떤 상품보다도 비싼 상품 찾기
- 기준은 전자기기 카테고리의 가장 저렴한 상품보다만 비싸면 됨
```sql
select name, price
from products
where price > ANY (select price from products where category = '전자기기');
```

### 문제 상황 2
- 전자기기 카테고리의 모든 상품보다 비싼 상품 찾기
```sql
select name, price
from products
where price > ALL (select price from products where category = '전자기기');
```

### 실무 팁: MIN, MAX 집계 함수와의 비교
- ANY, ALL보다 집계함수를 사용하는 것이 더 명확할 때가 많음
```sql
select name, price
from products
where price > (select min(price) from products where category = '전자기기');
```

```sql
select name, price
from products
where price > (select max(price) from products where category = '전자기기');
```

### 지금까지는 중간 정리
- (단일행, 단일열) - 스칼라
- (다중행, 단일열) - 다중행

- 지금부터는 단일행인데, 단일열인경우는?
	- 특정 상품과 판매자도 같고, 카테고리도 같은 다른 상품을 찾아라
	- **다중 컬럼 서브쿼리**를 사용

---

## 다중 컬럼 서브쿼리

### 개념
- select 절에 두 개 이상의 컬럼이 포함되는 경우
- where 절에서 여러 컬럼을 동시에 비교해야 할 때 매우 유용함

### 상황
- 쇼핑몰의 고객 네이트가 한 주문
- 이 주문과 동일한 고객이면서 주문 처리 상태도 같은 모든 주문을 검색

### 해결
```sql
select user_id, status from orders where order_id = 3; -- 하나의 행만 반환, 여러 열 반환

select * 
from orders
where (user_id, status) = (select user_id, status from orders where order_id = 3);
```

### 주의할 점
- 다중 컬럼 서브쿼리를 = 연산자와 함께 사용할 때는 단일 행 서브 쿼리와 마찬가지로 서브쿼리의 결과가 반드시 하나의 행을 반환해야함!!

### 다중 컬럼 서브 쿼리와 IN 연산자
- 여러 행일 경우 사용
- 각 고객별로 가장 먼저 한 주문의 주문ID, 사용자ID, 사용자이름, 제품이름, 주문날짜
```sql
-- 고객별 가장 처음에 했던 주문
select user_id, min(order_date)
from orders 
group by user_id;

select 
	o.order_id,
	o.user_id,
	u.name,
	p.name as product_name,
	o.order_date 
from orders o
join users u
	on o.user_id = u.user_id
join products p
	on o.product_id = p.product_id
where (o.user_id, o.order_date) in (
	select user_id, min(order_date)
	from orders 
	group by user_id
)
order by order_id;
```

---

## 상관 서브 쿼리 1
- 이전까지는 where 절에 서브쿼리를 사용하여, 메인쿼리와는 독립적으로 실행된 결과를 필터링 조건으로 사용하는 법을 학습

### 문제 상황
- 각 상품별로 자신이 속한 카테고리의 평균 가격 이상의 상품들을 찾아라
- 문제
	- 전체 평균이 아닌 자신이 속한 바로 그 카테고리의 평균 가격과 비교해야함
```sql
select * from products;

select avg(price) from products where category = '전자기기';
select avg(price) from products where category = '도서';
-- ... 각각의 행마다 비교하는 대상이 달라짐
```

### 상관 서브 쿼리의 개념
- 메인쿼리와 서브쿼리가 서로 **연관**을 맺고 동작하는 서브쿼리
- 서브쿼리가 독립적으로 실행될 수 없고, 메인쿼리의 값을 **참조**하여 실행되는 것이 특징

- 비상관 서브쿼리: 서브쿼리가 단 한 번 실행된후, 그 결과를 메인쿼리가 사용함
- 상관 서브쿼리:
	- 메인쿼리가 먼저 한 행을 읽음
	- 읽혀진 한 행의 값을 서브쿼리에 전달하여, 서브쿼리가 실행
	- 서브쿼리의 결과를 가지고 메인 쿼리의 where 조건을 판단
	- 메인쿼리의 다음 행을 읽고, 2-3번을 반복

### 문제 해결해보기
- 각 상품별로 자신이 속한 카테고리의 평균 가격 이상의 상품들을 찾아라
```sql
select *
from products p1
where
	price >= (select avg(price) 
				from products p2 
				where p2.category = p1.category);
```
- p1은 메인
- p2는 서브

### 단계별 동작
- 메인쿼리 실행
	- 한 행 씩 보면서 where를 가져오려고 시도
	- 첫 번째 행을 일단 가져옴
	- where에 비교를 하려고 보니 서브쿼리가 존재
	- 서브쿼리 동작
		- 일단 한 행을 가져왔으므로 p1의 categroy를 가져올 수 있음
		- 가져온 정보를 가지고 서브쿼리 실행 및 결과 반환
	- where과 비교해서 가져옴

--- 

## 상관 서브 쿼리 2

### 문세 상황
- product 테이블에는 있지만, orders 테이블에는 한 번도 등장하지 않은 상품
- 즉 "재고"로만 남아있는 상품을 제외하고, 실제 주문이 발생한 상품의 이름과 가격을 조회하고 싶음

- 일단 먼저 팔린 상품 확인
```sql
select product_id, name, price
from products;

select distinct product_id from orders o ;

select product_id, name, price
from products
where product_id in (select distinct product_id from orders o);

select product_id, name, price
from products
where product_id not in (select distinct product_id from orders o);
```
- 효율적인 방법
- 하지만 이건 실무에서는 항상 최선이 아닐 때가 있음

### EXISTS를 사용한 더 효율적인 방법
- IN 방식은 orders 테이블이 매우 클 경우, 성능 문제가 발생할 수 있음
- 이때, **EXISTS**를 사용하면 효율적으로 쿼리를 실행할 수 있음

- 연산
	- 서브쿼리 결과 행이 1개 이상이면 TRUE
	- 서브쿼리 결과 행이 0개이면 FALSE
```sql
select product_id, name, price
from products p
where not exists (
	select 1
	from orders o
	where o.product_id = p.product_id
);
```
- 그냥 값이 있냐 없냐만 판단 1개를 발견하는 순간 TRUE 반환 후 종료
- EXISTS는 결과 데이터가 무엇인지는 신경쓰지 않고, 행이 존재하는지 여부만 확인
	- 따라서 select 1같이 상수를 사용해 불필요한 데이터 조회를 피함

### 쿼리 실행 흐름
- EXISTS의 실행 흐름은 IN과 완전히 다름
- 메인쿼리
	- 한 행을 읽음
	- 이 행의 product_id를 가지고 서브쿼리 실행
		- 서브쿼리에서 첫번째 행을 찾자마자 더이상 탐색하지 않고 바로 TURE를 반환
	- 이 과정이 반복

### IN vs EXISTS: 실무에서는?
- IN
	- 서브쿼리를 먼저 실행해 결과 목록을 만든 후, 메인쿼리에서 사용
	- 서브쿼리 결과가 작을 때, 직관적이고 빠를 수 있음
	- orders 테이블 전체를 스캔해야 할 수 있음

- EXISTS
	- 메인쿼리의 각 행에 대해 서브쿼리를 실행하여 조건 확인
	- 상관 서브쿼리, 서브쿼리 테이블이 클 때 효율적
	- 조건을 만족하는 첫 행만 찾으면 스캔을 멈춤 

### 상관 서브쿼리와 성능
- 상관 서브쿼리는 복잡한 로직을 매우 직관적으로 표현할 수 있게 해주지만
- **성능에 매우 주의해야함**
- 메인 쿼리의 행수만큼 서브쿼리가 반복 실행될 수 있기 때문에
- **메인 쿼리가 다루는 데이터의 양이 많아지면 쿼리 전체의 성능이 급격히 저하될 수 있음**

- 많은 경우, 상관 서브쿼리는 join(특히 left join, group by)으로 동일한 결과를 얻도록 재작성할 수 있으며, 
- **데이터베이스 옵티마이저가 join을 더 효율적으로 처리하는 경우가 많음**

- 하지만 특정 상황에 따라서 exists가 더 효율적으로 동작하는 상황도 있음

### 다음으로
- 지금까지는 where절에 서브쿼리를 사용하는 것을 배움
- select 절 안으로 가져온다면?
	- **각 행마다 새로운 정보를 계산해서 보여주는, 전혀 다른 방식의 활용이 가능해짐**
	- **스칼라 서브 쿼리**

---

## SELECT 서브 퀴리
- select절에서 사용하면 필터링 조건이 아닌, 나의 컬럼으로 동작하게 됨
- 참고로 select 절에서는 단일 값(하나의 행, 하나의 열)을 반환하는 **스칼라 서브쿼리**를 사용해야만 함

### 비상관 서브 쿼리
- 바깥쪽 쿼리에 전혀 영향을 받지 않는 독립적인 서브 쿼리
- 문제 상황: 전체 상품의 평균 가격을 모든 행에 함께 표시해서 개별 상품 가격이 평균과 얼마나 차이 나는지 비교해보고 싶음
- 이 경우, 전체 상품의 평균 가격은 어떤 특정 상품 행에 종속되는 값이 아니라, 모든 상품에 대해 동일하게 적용되는 고정값
```sql
select avg(price) from products;

select name, price, (select avg(price) from products) as avg_price
from products;
```

### 상관 서브 쿼리
- 문제 상황: 전체 상품 목록을 조회하면서, 각 상품별로 총 몇 번의 주문이 있었는지 "총 주문 횟수"를 함께 보여주고 싶음
```sql
select * from products;

select count(*) from orders o where o.product_id = 1;
select count(*) from orders o where o.product_id = 2;
-- ...

select 
	p1.product_id,
	p1.name,
	p1.price,
	(select count(*) from orders o where o.product_id = p1.product_id) 
from products p1;
```

### 성능에 주의하라
- 스칼라 서브쿼리는 이처럼 join으로 표현하기 복잡한 로직을 직관적으로 표현할 수 있게 해주지만, 강력한 만큼 주의해서 사용해야함
- **성능 저하** 가능성
- count(*)가 100만번이나 실행될 수 있음(products가 100만개면)
```sql
select p.product_id, p.name, p.price, count(o.order_id) as order_count
from products p
left join orders o on p.product_id = o.product_id
group by p.product_id, p.name, p.price;
```

---

## 테이블 서브 쿼리 (인라인 뷰)

### 개념
- 실행 결과가 마치 하나의 독립된 가상 테이블처럼 사용되기 때문에 테이블 서브쿼리라고 함
- 우리가 지금가지 배운 복잡한 select문(집계, 그룹핑, 조인 등)의 결과를 하나의 명확한 데이터 집합으로 먼저 만들어 놓고, 그 집합을 대상으로 다시 한번 select를 할 수 있게 해준다는 점

### 문제 상황
- 각 상품 카테고리별로, 가장 비싼 상품의 이름과 가격을 조회하고 싶음
```sql
select category, max(price)
from products 
group by category; -- name을 사용할 수 없는 문제!!

select name, category, price from products
where category = '전자기기';

select category, max(price) as max_price
from products 
group by category; -- 가상 테이블이라고 생각

select p.category, p.name, p.price
from products p
join (
	select category, max(price) as max_price
		from products 
		group by category
) pv
	on p.category = pv.category and p.price = pv.max_price; 
```
- 핵심: 가상 테이블이랑 조인!!!

### 쿼리 실행 흐름
- 데이터베이스는 from절의 서브쿼리를 먼저 실행하여, 임시 테이블을 메모리에 생성
- 이것을 가지고 join
- **group by**의 함정을 피하고, 각 카테고리별로 가장 비싼 상품의 정보를 찾을 수 있음

---

## 서브쿼리 vs JOIN
- 어떤 문제는 두 가지 방법으로 풀 수 있음

### 문제 상황
- 서울에 거주하는 모든 고객들의 주문 목록을 조회해라

### 서브쿼리 사용
- 우리의 사고 흐름과 비슷함
```sql
select user_id from users where address like '서울%';

select * from orders o where o.user_id in (
	select user_id from users where address like '서울%');
```

### JOIN 사용
```sql
select
	o.user_id,
	o.order_id,
	o.product_id,
	o.order_date
from orders o
join users u
	on o.user_id = u.user_id
where u.address like '서울%';
```
- 서브쿼리를 사용했을때와 동일함

### 성능 vs 가독성
- 성능
	- join이 서브쿼리보다 성능이 더 좋거나 최소한 동일한 경우가 많음

- 이유
	- 비밀은 바로 쿼리 옵티마이저에 있음
	- join 구문은 옵티마이저에게 더 많은 정보를 제공함
	- 전체 그림을 미리 보여주기 때문에, 인덱스를 어떻게 활용하고 어떤 테이블을 먼저 읽을지 등 가장 효율적인 실행 계획을 선택할 수 있음

- 하지만
	- 요즘 데이터베이스 쿼리 옵티마이저는 서브쿼리를 내부적으로 최적의 조인 구문으로 자동 변환해주는 경우가 많음
	- 따라서 위 예제의 두 쿼리는 동일한 성능을 낼 확률이 높음
	- 쿼리 실행 계획을 확인하는 것이 좋음!!!

### 가독성
- 서브쿼리는 논리적인 단계를 명확히 구분해주어 복잡한 로직을 더 이해하기 쉽게 만들어주는 경우가 많음

### 최종 결론
- 정답은 없음

1. 조인을 우선적으로 고려
2. 조인으로 표현하기 너무 어렵거나, 서브쿼리의 가독성이 훨씬 좋다면 서브쿼리를 사용하자
3. EXISTS를 활용하자
	- IN 서브쿼리 대안으로 EXISTS라는 서브쿼리 연산자를 활용
4. 성능이 의심될 때는 반드시 측정해라
	- EXPLAIN 같은 도구를 활용해서 쿼리 실행 계획을 반드시 확인하자!!! 

---

## 문제와 풀이

### 가장 비싼 상품 조회하기
```sql
select 
	p.product_id,
	p.name,
	p.price
from 
	products p
where p.price = (select max(price) from products);
```

### 동일 삼품 주문 정보 조회하기
```sql
select 
	o.order_id,
	o.user_id,
	o.order_date
from orders o
where o.product_id = (
	select product_id from orders where order_id = 1) and o.order_id != 1;
```

### 고객별 총 주문 횟수 조회하기
```sql
select count(*)
from orders o 
where o.user_id = 6;

select 
	u.name as `고객명`,
	(select count(*) from orders o where o.user_id = u.user_id) as `총주문횟수`
from users u
order by u.user_id;
```
