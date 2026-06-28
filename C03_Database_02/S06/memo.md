# S06. CASE 문

## CASE 문 기본 1

### 개념
- 데이터 자체를 동적으로 가공하고, 새로운 의미를 부여하는 기술

### 문제 상황
- 상품 목록을 조회하는데, 그냥 가격만 보여주지 말고, 가격대에 따라 "고가", "중가", "저가"를 함께 표시
    - 10만원 이상: 고가
    - 3만원 이상 10만원 미만: 중가
    - 3만원 미만: 저가

- 물론 애플리케이션 코드에서 처리할 수 있음
- 간단한 보고서나 데이터 분석을 할 때, 쿼리 하나로 할 수 있음

### 단순 CASE 문
- 특정 하나의 컬럼이나 표현식의 값에 따라 결과를 다르게 하고 싶을 때 사용함
```sql
CASE 비교대상_컬럼_또는_표현식
    WHEN 값 THEN 결과
    ...
    ELSE 그_외의_경우_결과
END
```
- ELSE를 생략했는데, 모든 WHEN 조건이 거짓이면 **NULL**을 반환함
- 실행 순서:
    - 위에서 아래로 가장 먼저 일치하는 WHEN절을 만나는 순간 그 THEN의 결과를 반환하고 즉시 평가 종료

### 예시
```sql
select 
	order_id,
	user_id,
	product_id,
	quantity,
	status,
	case status
		when 'PENDING' then '주문 대기'
		when 'COMPLETED' then '결제 완료'
		when 'SHIPPED' then '배송'
		when 'CANCELLED' then '주문 취소'
		else '알 수 없음'
	end
from orders;
```

### 검색 CASE 문
- 하나의 특정 값을 비교하는 대신, when절에 독립적인 조건식을 사용하여 복잡한 논리를 구현할 때 사용

### 예제
- 상품 각격에 따라 등급 표시하기
```sql
select
	name,
	price,
	case
		when price >= 100000 then '고가'
		when price >= 30000 and price < 100000 then '중가'
		else '저가'
	end
from products;
```

---

## CASE 문 기본 2

### 검색 CASE 문 사용시 주의사항: WHEN 절의 순서
- 결론: 조건을 잘 배치하자!!

### CASE 문과 사용 위치
- select뿐 아니라, order by, group by, where 절 등 다양한 SQL 구문과 함께 사용될 수 있음
- 상품을 고가, 중가, 저가 순서로 정렬하고 싶다면 order by절에 case문을 사용할 수 있음
```sql
select
    name,
    price,
    case
        when price >= 100000 then '고가'
        when price >= 30000 and price < 100000 then '중가'
        else '저가'
    end as price_label
from 
    products
order by
    case 
        when price >= 100000 then 1
        when price >= 30000 and price < 100000 then 2
        else 3
    end asc,
    price desc;
```
- 진정한 힘은 특히 집계 함수나 group by에서 사용될 때, 진가를 발휘함!!

---

## CASE 문 - 그룹핑

### 문제 상황
- 고객들을 출생 연대에 따라 1990년대생, 1980년대생, 그 이전 출생으로 분류하고, 각 그룹에 고객이 총 몇 명씩 있는지 알고 싶음
    - 분류: CASE를 활용해서 각 고객에게 출생 라벨을 붙여줌
    - 집계: 라벨을 기준으로 그룹핑하고 고객 수를 셈
```sql
select
	case
		when birth_date between '1990-01-01' and '2000-01-01' then '1990년대생'
		when birth_date between '1980-01-01' and '1990-01-01' then '1980년대생'
		else '그 이전 출생'
	end as birth_label,
	count(user_id)
from users
group by
	case
		when birth_date between '1990-01-01' and '2000-01-01' then '1990년대생'
		when birth_date between '1980-01-01' and '1990-01-01' then '1980년대생'
		else '그 이전 출생'
	end;
```
- 아 너무 긴데?

```sql
select
	case
		when birth_date between '1990-01-01' and '2000-01-01' then '1990년대생'
		when birth_date between '1980-01-01' and '1990-01-01' then '1980년대생'
		else '그 이전 출생'
	end as birth_label,
	count(user_id)
from users
group by
	birth_label; -- select의 별칭으로 사용할 수 있음
```
- MySQL이 허용해주는 기능

---

## CASE 문 - 조건부 집계 1
- CASE문이 집계 함수 안으로 들어옴

### 문제 상황
- 하나의 쿼리로, 전체 주문 건수와 함께 결제 완료, 배송, 주문 대기 상태의 주문이 각각 몇 건인지 별도의 컬럼으로 나누어보고 싶음

- UNION으로 처리
```sql
select 'Total' as category, count(*) as total_orders from orders

union all

SELECT 
	status, count(*)
FROM orders 
group by status;
```

- 서브 쿼리 활용
```sql
select 
    (select count(*) from orders) as total_orders,
    (select count(*) from orders where status = 'COMPLETED') as completed_count,
    (select count(*) from orders where status = 'SHIPPED') as shipped_count,
    (select count(*) from orders where status = 'PENDING') as pending_count
-- MySQL에서 from 절을 생략해도 됨
```
- 서브 쿼리 활용 문제점
    - 지금 하나의 쿼리라고 생각하면 안됨
    - select 안에서 select을 4번이나 추가적으로 날림
    - 심각한 성능 문제가 있음

- 결론: CASE 문을 집계 함수 안에 넣는 조건부 집계 기법이 필요함

---

## CASE 문 - 조건부 집계 2

### CASE를 품은 집계 함수
- count안에 case를 넣을 수 있음
```sql
count(case when status = 'COMPLETED' then 1 end)
```
    - status가 COMPLETED이면 case문은 숫자 1을 반환
    - 그 외의 경우(ELSE)가 없으므로 case문은 null을 반환
    - 결과적으로 count함수는 status가 COMPLETED인 행의 개수만 세게 됨

- sum
```sql
sum(case when status = 'COMPLETED' then 1 else 0 end)
```
    - status가 COMPLETED이면 case문은 1을, 그외에는 0을 반환
    - sum함수가 1과 0들을 모두 더하면, 그 합계는 결국 COMPLETED 상태인 주문의 총 개수가 됨

### 실습 1: 전체 주문 상태 요약하기
```sql
select
	count(*) as total_orders,
	sum(case when status = 'COMPLETED' then 1 else 0 end) as completed_count,
	sum(case when status = 'SHIPPED' then 1 else 0 end) as shipped_count,
	sum(case when status = 'PENDING' then 1 else 0 end) as pending_count
from
 	orders; -- 이건 딱 한 번만 읽음
```

### 실습 2: Group By와 함께 사용하기 (피벗 테이블)
- 한 단계 더 나아가 상품 카테고리 별로, 상태별 주문 건수를 집계하라
```sql
select
	p.category,
	count(*) as total_orders,
	sum(case when o.status = 'COMPLETED' then 1 else 0 end) as completed_count,
	sum(case when o.status = 'SHIPPED' then 1 else 0 end) as shipped_count,
	sum(case when o.status = 'PENDING' then 1 else 0 end) as pending_count
from
 	orders o
join products p
	on o.product_id = p.product_id 
group by p.category;
```

- 이런걸 이제 자주 사용하게 될텐데 이걸 매번 작성하거나 복붙하는 것은 힘듦
    - 마법 같은 가상 테이블, VIEW에 대해서 학습

---

## 문제와 풀이

### 상품 카테고리 영문으로 표시하기
```sql
select 
    name,
    category,
    case category
        when '전자기기' then 'Electronics'
        when '도서' then 'Books'
        when '패션' then 'Fashion'
        else 'etc
    end as category_english
from products;
```

### 주문 수량에 따른 분류 및 정렬
```sql
select
    o.order_id,
    o.quantity,
    case
        when o.quantity >= 2 then '다량 주문'
        when o.quantity = 1 then '단건 주문'
    end
from 
    orders o
order by
    case
        when o.quantity >= 2 then 1
        when o.quantity = 1 then 2
    end;
```

### 재고 수준별 상품 수 집계하기
```sql
select
    case
        when stock_quantity  >= 50 then '재고 충분'
        when stock_quantity >= 20 and stock_quantity < 50 then '재고 보통'
        else '재고 부족'
    end as stock_level,
    count(*) as product_count
from
    products
group by 
    stock_level;
```

### 사용자별 카테고리 주문 건수 피벗 테이블 
```sql
select
    u.name as user_name,
    count(o.order_id) as total_orders, 
    sum(case when p.category = '전자기기' then 1 else 0 end) as electronics_orders,
    sum(case when p.category = '도서' then 1 else 0 end) as book_orders,
    sum(case when p.category = '패션' then 1 else 0 end) as fashion_orders
from users u
left join orders o
    on u.user_id = o.user_id
left join products p
    on o.product_id = p.product_id
group by
	u.name;
```