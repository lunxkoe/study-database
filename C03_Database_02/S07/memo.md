# S07. View

## View 소개

### 문제 상황
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
- 이 쿼리를 그냥 복사해서 사용하면 발생할 수 있는 문제
    - 복합성
    - 재사용성
    - 보안: 사용자는 orders와 products에 직접 접근할 수 있는(select) 권한이 있어야함
    
- 해결
    - 데이터베이스에 하나의 "바로 가기"처럼 저장해두고 필요할 때마다 사용

### 뷰의 개념
- 실제 데이터를 가지고 있지 않은 가상의 테이블
- 그 자체로 데이터를 저장하는 테이블이 아님
- 단지 복잡합 SELECT 쿼리문 자체를 저장하고 있음
- SELECT * FROM 나의_바로가기_뷰;라는 간단한 명령어로 복잡한 쿼리의 결과를 얻을 수 있음

### 뷰의 동작 원리
- v_category_order_status라는 뷰를 조회하면
    - select * from v_category_order_status;라는 쿼리를 실행

> 뷰는 테이블을 저장하지 않기 때문에, 조회마다 최신 상태의 원본 테이블을 기준으로 쿼리가 실행됨

---

## 뷰 생성, 조회, 수정, 삭제

### 뷰 생성하기: CREATE VIEW
```sql
create view 뷰이름 as select 쿼리문; 
```

### 뷰 조회하기: SELECT
```sql
select * from v_category_order_status;
```

### 뷰 수정하기: ALTER VIEW
```sql
alter view v_category_order_status as
select
	p.category,
	sum(p.price * o.quantity) as total_sales, -- 매출액 컬럼 추가
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

### 뷰 삭제하기: DROP VIEW
```sql
drop view v_category_order_status;
```

### 중요한 사실
- 뷰를 삭제해도 원본 테이블 데이터는 단 하난도 손상되거나 변경되지 않음

---

## 뷰의 장점과 단점

### 장점
- 편리성과 재사용성
- 보안성
    - 특정 컬럼 숨기기
    - 특정 행만 노출하기
- 논리적 데이터 독립성
    - 일종의 추상화 계층

### 단점
- 성능 문제
    - 간단한 쿼리 뒤에서 무거운 쿼리가 숨어있을 수 있음
    - 뷰 안에 뷰를 넣는 경우도 있음: 성능 저하의 주요 원인
- 업데이트 제약
    - 수정 불가
    - 수정 가능: 뷰가 오직 하나의 기본 테이블만 참조하고, 뷰의 모든 컬럼이 기본 테이블의 실제 컬럼을 직접 참조하는 경우에는 뷰에 데이터를 추가/수정할 수 있음
        - **원본 테이블의 값이 변경됨**
    - 실무 규칙: 뷰는 기본적으로 **조회용**

---

## 문제와 풀이

### 고객의 기본 정보만 보여주는 뷰 생성하기
```sql
drop view if exists v_customer_email_list;

create view v_customer_email_list as
select
    user_id,
    name,
    email
from users;

select * from v_customer_email_list;
```

### 주문 상세 정보를 통합한 뷰 생성하기
```sql
drop view if exists v_order_summary;

create view v_order_summary as
select 
    o.order_id,
    u.name as `고객명`,
    p.name as `상품명`,
    o.quantity,
    o.status
from orders o
join users u
    on o.user_id = u.user_id
join products p
    on o.product_id = p.product_id;

select * from v_order_summary;
```

### '전자기기' 카테고리 매출 분석 뷰 생성하기
```sql
drop view if exists v_electronics_sales_status;

create view v_electronics_sales_status as
select 
    p.category,
    count(o.order_id) as total_orders,
    sum(o.quantity * p.price) as total_sales
from 
    orders o
join
    products p
    on o.product_id = p.product_id
where
    p.category = '전자기기'
group by
    p.category;

select * from v_electronics_sales_status;
```

### 기존 뷰에 정보 추가하여 수정하기
```sql
alter view v_electronics_sales_status as
select 
    p.category,
    count(o.order_id) as total_orders,
    sum(o.quantity * p.price) as total_sales,
    avg(o.quantity * p.price) as average_order_value
from 
    orders o
join
    products p
    on o.product_id = p.product_id
where
    p.category = '전자기기'
group by
    p.category;

select * from v_electronics_sales_status;
```