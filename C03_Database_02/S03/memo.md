# S03. 조인2 - 외부 조인과 기타

## 외부 조인 1

### 외부 조인의 필요성 대두
- 쇼핑몰에 가입은 했지만, 아직 한 번도 주문하지 않은 고객
- 출시했지만, 아직 단 한 번도 팔리지 않은 상품
- 주문이 되지 않아, orders에 없음

### 실습: 한 번도 주문하지 않은 고객 찾기
- 무식한 방법
    - users, orders를 각각 조회 후 비교해서 직접 찾기
    - inner join을 활용해서 없는 user_id를 찾음

### 외부 조인의 필요성
- 내부 조인으로는 주문 기록이 없는 고객을 찾을 수 없음
- 외부 조인을 사용하면 한쪽 테이블에만 존재하는 데이터도 결과에 포함시킬 수 있음
- **한쪽에는 데이터가 있지만, 다른 한쪽에는 없는 데이터까지 모두 포함해서 보고 싶을 때 사용하는 기술이 바로 외부 조인**

### 외부 조인 필요성 및 개념
- 조인은 본질적으로 두 테이블의 집합에 대한 이야기
- 외부 조인은 두 테이블을 조인할 때, 특정 테이블의 데이터는 on 조건이 맞지 않더라도 모두 결과에 포함시키는 방법
- 이때 기준이 중요함
    - left outer join: A만 포함시킴
    - right outer join: B만 포함시킴
    - full outer join: A, B를 전부 포함시킴
        - **실무에서 잘 사용하지 않고, MySQL은 지원하지 않음**

### left outer join vs right outer join
- 핵심: 기준 테이블을 정하는 것!!
    - left outer join: 왼쪽 테이블을
    - right outer join: 오른쪽 테이블을 
    - 기준으로 삼음

> outer는 생략 가능

### left join
- 왼쪽(from 절)에 있는 테이블이 기준이 됨
- 일단 왼쪽 테이블의 모든 데이터를 결과에 포함
- 그 다음, on절에 맞는 데이터를 오른쪽 테이블에서 찾아 옆에 붙여줌
- 만약 오른족 테이블에 짝이 맞는 데이터가 없다면, 그 자리는 null값으로 채워짐

### right join
- 오른쪽(join 절)에 있는 테이블이 기준이 됨
- 일단 오른쪽 테이블의 모든 데이터를 결과에 포함
- 그 다음, on절에 맞는 데이터를 왼쪽 테이블에서 찾아 붙여줌
- 마찬가지로 왼쪽 테이블에 짝이 맞는 데이터가 없다면, 그 자리는 null로 채워짐

### 실습 - Left JOIN으로 한번도 주문하지 않은 고객 찾기
- 고객이 기준
```sql
select
    u.user_id,
    u.name,
    u.email
from users u
left join orders o
    on u.user_id = o.user_id
where o.user_id is null -- 주문한 내역이 없는 고객에 대한 조건
order by u.user_id;
```

---

## 외부 조인 2

### 실습 2: LEFT JOIN으로 "단 한 번도 팔리지 않은 상품 찾기"
- products가 기준이 되어야함(from 절)
```sql
select 
	p.product_id,
	p.name
from products p
left join orders o
	on p.product_id = o.product_id 
where o.order_id is null;
```

### 실습 3: RIGHT JOIN으로 단 한 번도 팔리지 않은 상품 찾기
```sql
select
	p.product_id,
	p.name
from orders o
right join products p
	on o.product_id = p.product_id
where o.order_id is null;
```
- orders를 from에 두고 products를 join으로 바꿈

### LEFT vs RIGHT 조인 선택 - 결론: LEFT
- 서로 위치를 바꾸어 사용할 수 있음
- 다이어그램을 보면 됨!!
- 즉, 원리 자체는 동일함 (테이블을 바꾸게 된다면!!)

> 실무에서는 LEFT를 더 많이 사용함
- 기준이 되는 테이블을 보통 from에 적고 left join으로 붙이는 것이 더 직관적임
- right join은 테이블의 순서를 바꾸면 언제나 left join으로 동일하게 표현할 수 있음

---

## 조인의 특징
- 두 테이블을 조인할 때 어떤 경우에는 행이 더 늘어나고, 어떤 경우에는 행이 늘어나지 않고 그대로인 경우가 있음
- 데이터베이스를 다루는데 있어 정말 중요함

### 조인에서 데이터가 늘어나는 경우
- 기준으로 삼는 테이블의 한 행이 다른 쪽 테이블의 여러 행과 연결될 수 있다면, 결과적으로 전체 행 수는 늘어남
- 반대로 한 행이 다른 쪽 테이블의 단 하나의 행과 연결되거나, 아무 행과도 연결되지 않는다면 행의 수는 늘어나지 않음

- 그렇다면 언제?
    - 기본키와 외래키의 관계를 제대로 이해해야함

### 기본키 / 외래키
- 기본키: 각 행을 고유하게 식별하는 값
    - PK는 테이블 내에서 절대로 중복될 수 없음

- 외래키: 다른 테이블의 PK를 참조하는 값
    - FK는 테이블 내에서 여러 번 중복되서 나타날 수 있음

- 부모-자식 관계
    - 부모 테이블: PK를 가지고 있는 테이블
    - 자식 테이블: FK를 통해 부모 테이블을 참조하는 테이블

### 그래서 도대체 언제
- 자식 -> 부모 조인 (FK -> PK 참조): **행의 개수가 늘어나지 않음**

- 부모 -> 자식 조인 (PK -> FK 참조): **행의 개수가 늘어날 수 있음**

### 실습
- 자식 -> 부모 조인: 행의 개수가 늘어나지 않음
```sql
select order_id, product_id, user_id
from orders
where user_id = 1;

select *
from 
	orders o -- 자식에서
join 
    users u -- 부모참조
	on o.user_id = u.user_id
where o.user_id = 1;
```

- 부모 -> 자식 조인: 행의 개수가 늘어날 수 있음 (아래는 늘어남)
```sql
select user_id, name, email
from users
where user_id = 1;

select *
from 
    users u -- 부모에서
join  
    orders o -- 자식참조
	on u.user_id = o.user_id
where o.user_id = 1; 
```

- 자식 -> 부모 조인
```sql
select order_id, product_id, user_id
from orders; -- 7건

select
	o.order_id,
	o.product_id,
	o.user_id as orders_user_id,
	u.user_id as users_user_id,
	u.name,
	u.email
from 
	orders o	
join	
	users u on o.user_id = u.user_id; -- 7건 (orders의 행의 수가 늘어나지 않음)
```

- 부모 -> 자식 조인
```sql
select user_id, name, email
from users; -- 6건

select
	o.order_id,
	o.product_id,
	o.user_id as orders_user_id,
	u.user_id as users_user_id,
	u.name,
	u.email
from 
	users u	
join	
	orders o on o.user_id = u.user_id; -- 7건 (users의 행의 수가 늘어남)
```

### 결론: 언제 행이 늘어나고 언제 그대로인가?
- 행 개수 유지: 자식에서 부모로 조인할 때 (to-One)
    - from 자식 테이블 join 부모 테이블의 단 하나의 행과 매칭됨
    
- 행 개수 증가 가능: 부모에서 자식으로 조인할 때 (to-Many)
    - from 부모 테이블 join 자식 테이블의 여러 행과 매칭될 수 있음

### 왜 중요할까? 문제 발생 예
- 고객과 그들의 주문 정보를 보기 위해서 from users join orders를 수행
- 여기서 고객 수를 세기 위해서 그냥 count(u.user_id)를 실행하면?
    - 전체 고객 수가 나오는 것이 아니라, 그냥 행의 수가 나와버림
    ```sql
    select
        count(distinct u.user_id)
    from 
        users u
    join 
        orders o
        on u.user_id = o.user_id;
    ```

- 조인을 잘 이해해야지, 집계 함수를 올바르게 사용할 수 있음

### 조인의 유연성
- 조인은 주로 기본 키와 외래 키를 써서 테이블을 연결하는 것이 가장 일반적이고 중요한 방식
- 그러나 조인의 핵심 원리는
    - **두 테이블의 특정 열의 값이 같은가임**
    - **데이터 타입만 같다면** 어떤 열이든 조인 조건으로 쓸 수 있음(PK-FK 관계가 아니여도 됨)
        - 데이터 타입을 변경 후에 해도 됨

### PK-FK 조인이 중요한 이유
- 조인은 유연하지만
- 실무에서는 데이터의 정합성과 일관성을 위해 대부분 PK-FK를 사용함

--- 

## 셀프 조인(SELF JOIN)

### 문제 상황
- 각 직원의 이름과 바로 위 직속 상사의 이름을 나란히 함께 출력하려면 어떻게 해야할까?
- 직원 테이블을 보면, manager_id가 employee_id인 것을 확인할 수 있음
- 이런 관계를 자기 참조 관계라고 하고
    - 이런 구조의 데이터를 한 번의 쿼리로 조회하기 위해 사용하는 기술이 바로 self join

### 개념과 원리
- 하나의 테이블을 자기 자신과 조인하는 기법을 말함
- 핵심 원리
    - **테이블 별칭**
    - e: 모든 직원 목록
    - m: 모든 상사의 목록

### 실습: 직원-상사 목록 만들기 
```sql
select 
	e.name as `직원`,
	m.name as `상사`
from employees e
left join employees m
	on e.manager_id = m.employee_id;
```

---

## CROSS 조인 - on 절이 없는 조인
- 지금까지 inner, outer, self 조인은 모두 on이라는 연결고리를 통해 테이블에 
- **이미 존재하는 관계**를 찾아내는 작업
- 그런데 만약, 애초에 짝이나 관계가 없는 두 집단을 가지고 가능한 모든 조합을 구해야한다면?

### 문제 상황
- 기본 티셔츠 판매
- 사이즈는 S, M, L, XL
- 색상은 Red, Blue, Black
- 재고 관리를 위해 모든 사이즈와 색상 조합을 담은 상품 마스터 데이터를 미리 생성하고 싶다면
    - size, colors 테이블 사이에는 미리 정해진 연결고리가 없음!!
    - 이 둘을 강제로 조인해야함!!
    - **이때 사용하는 것이 cross join**

### cross join의 개념과 카테시안 곱
- 한쪽 테이블의 모든 행을 다른 쪽 테이블의 모든 행과 하나씩 전부 연결하는, 가장 단순한 조인 방식
- 연결 고리가 없기 때문에 **on절을 사용하지 않음**
- 카테시안 곱
    - A: m행
    - B: n행
    - AxB: m x n행

```sql
select * from sizes; -- 3개
select * from colors; -- 4개

select *
from sizes s
cross join colors c; -- 3 * 4 = 12개

select 
	concat('기본티셔츠-', c.color, '-', s.size) as product_name,
	s.size,
	c.color
from sizes s
cross join colors c;
```
- 이 결과를 새로운 테이블에 insert 하면, 상품 마스터 데이터를 손쉽게 구축할 수 있음

### INSERT INTO ... SELECT로 상품 옵션 마스터 테이블 만들기
- 데이터를 한 번에 대량으로 삽입할 때 insert into ... select 구문을 사용
- product_options 테이블을 만들 것
    - option_id
    - product_name
    - size
    - coloer

```sql
insert into product_options (product_name, size, color)
	select 
		CONCAT('기본티셔츠-', c.color, '-', s.size) AS product_name,
		s.size,
		c.color
	from
		sizes s
	cross join
		colors c;
```
- 이처럼 단순히 데이터 조회를 넘어서 새로운 데이터 세트를 만들거나 초기 시스템을 구축할 때 유용하게 활용할 수 있음

### 실무에서 치명적인 주의사항
- 모든 경우의 수를 만들어주기 때문에 유용하지만, 실무에서는 아주 신중하게 사용
- 결과의 행의 수가 기하급수적으로 늘어날 수 있음
- 디비 서버가 다운될 수도 있음

---

## 조인 종합 실습

### 최종 미션
- 2025년 6월 서울에 거주하는 고객이 주문한 모든 내역에 대해, 고객 이름, 고객 이메일, 주문 날짜, 주문한 상품명, 상품 가격, 주문 수량을 포함하는 상세 보고설르 최신 주문 순으로 작성하라

### 문제 분석 및 해결 전략
- 필요한 정보
    - 고객 이름, 고객 이메일 -> users 테이블
    - 주문 날짜, 주문 수량 -> orders 테이블
    - 주문한 상품명, 상품 가격 -> products 테이블
    - 주문 내역에 대한 것이므로 주문 테이블을 기준으로 시작하는 것이 좋아보임

- 필요한 테이블
    - users, orders, products 세 개의 테이블이 모두 필요함

- 필터링 조건
    - 조건1: 2025년 6월에 이루어진 주문 -> orders 테이블
    - 조건2: 서울에 거주하는 고객의 주문 -> users 테이블

- 정렬 순서
    - 최신 주문 순서 -> orders table의 order_date 내림차순

```sql
select 
	u.name as `고객 이름`,
	u.email as `고객 이메일`,
	o.order_date as `주문 날짜`,
	o.quantity as `주문 수량`,
	p.name as `주문한 상품명`,
	p.price as `상품 가격`
from orders o
join users u
	on o.user_id = u.user_id
join products p
	on o.product_id = p.product_id
where 
	(o.order_date >= '2025-06-01' and o.order_date < '2025-07-01') and u.address like '서울%'
order by
	o.order_date desc;
```
- 주의사항: between을 사용하면 안되는 이유(엄밀하게 하면 사용해도 되긴한데)
    - 2025-06-01은 00:00:00 시간을 포함하고 있음

### 조인으로 해결할 수 없는 문제
- 가장 비싼 상품을 주문한 고객은 누구일까?
    - 일단 먼저 상품 중에서 가장 비싼 상품의 가격을 알아내고
    - 가격으로 상품을 찾은 뒤
    - 그 상품을 주문한 고객을 찾아야함
    - **서브 쿼리 활용**

---

## 문제와 풀이 1

### 특정 카테고리의 미판매 상품 찾기
```sql
select
	p.name,
	p.price 
from 
    products p
left join 
    orders o
	on o.product_id = p.product_id
where 
    o.order_id is null and p.category = '전자기기';
```

### 고객별 주문 횟수 구하기 (주문 없는 고객 포함)
```sql
select 
	u.name,
	count(o.order_id) -- null이면 count를 0으로 셈
from 
	users u
left join 
	orders o
	on u.user_id = o.user_id
group by
	u.user_id, u.name -- id로 하는 것이 정배, select에서 사용할 수 있게 u.name 추가
    -- (user_id, name)이 같은 걸 기준으로 함
order by
	u.name;	
```

### RIGHT JOIN으로 주문 없는 고객 찾기
```sql
select 
    u.name, 
    u.email
from 
    orders o
right join 
    users u
	on o.user_id = u.user_id
where 
    o.order_id is null;
```

### 고객별 주문 상품 목록 조회하기
```sql
select
	u.name as user_name,
	p.name as product_name
from
	users u
left join 
	orders o
	on u.user_id = o.user_id
left join 
	products p
	on o.product_id = p.product_id
order by 
	u.name, p.name;
```

---

## 문제와 풀이 2

### 특정 상사의 부하 직원 찾기
```sql
select 
	e2.employee_id ,
	e2.name,
	e1.employee_id,
	e1.name as manager_name
from
	employees e1
join 
	employees e2
	on e1.employee_id = e2.manager_id
where e1.name = '최과장';
```
- 특정 상사의 부하직원을 찾는 것이므로 
- 상사가 기준이 되는게 좋음

### 모든 상품 옵션 조합에 재질 추가하기
```sql
select
	concat(p.name, '-', c.color, '-', s.size, '-', m.material) as product_full_name
from 
	products p 
cross join 
	sizes s 
cross JOIN 
	colors c 
cross join 
	materials m
order by
	s.size, c.color, m.material;
```

### 특정 고객의 주문 내역 상세 조회하기
```sql
select
	u.name,
	p.name, 
	o.order_date, 
	o.quantity 
from 
	users u
join
	orders o
	on u.user_id = o.user_id
join 
	products p
	on o.product_id = p.product_id
where
	u.name = '네이트'
order by 
	o.order_date desc;
```

### 서울 지역 고객의 총 주문 금액 계산하기
```sql
select
	u.name as customer_name,
	sum(p.price * o.quantity) as total_spent
from 
	users u
join 
	orders o
	on u.user_id = o.user_id
join  
	products p
	on o.product_id = p.product_id
where 
	u.address like '서울%'
group by
	u.user_id, u.name
order by 
	sum(p.price * o.quantity) desc;
```
