# S06. 조인

## 실습 데이터 준비
```sql
-- 데이터베이스가 존재하지 않으면 생성
CREATE DATABASE IF NOT EXISTS my_shop2;
USE my_shop2;
-- 테이블이 존재하면 삭제 (실습을 위해 초기화)
DROP TABLE IF EXISTS orders;
DROP TABLE IF EXISTS users;
DROP TABLE IF EXISTS products;
DROP TABLE IF EXISTS employees;
DROP TABLE IF EXISTS sizes;
DROP TABLE IF EXISTS colors;
-- 고객 테이블 생성
CREATE TABLE users (
 user_id BIGINT AUTO_INCREMENT,
 name VARCHAR(255) NOT NULL,
 email VARCHAR(255) NOT NULL UNIQUE,
 address VARCHAR(255),
 birth_date DATE,
 created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
 PRIMARY KEY (user_id)
);
-- 상품 테이블 생성
CREATE TABLE products (
 product_id BIGINT AUTO_INCREMENT,
 name VARCHAR(255) NOT NULL,
 category VARCHAR(100),
 price INT NOT NULL,
 stock_quantity INT NOT NULL,
 PRIMARY KEY (product_id)
);
-- 주문 테이블 생성
CREATE TABLE orders (
 order_id BIGINT AUTO_INCREMENT,
 user_id BIGINT NOT NULL,
 product_id BIGINT NOT NULL,
 order_date DATETIME DEFAULT CURRENT_TIMESTAMP,
 quantity INT NOT NULL,
 status VARCHAR(50) DEFAULT 'PENDING',
 PRIMARY KEY (order_id),
 CONSTRAINT fk_orders_users FOREIGN KEY (user_id)
 REFERENCES users(user_id),
 CONSTRAINT fk_orders_products FOREIGN KEY (product_id)
 REFERENCES products(product_id)
);
-- 직원 테이블 생성 (SELF JOIN 실습용)
CREATE TABLE employees (
 employee_id BIGINT AUTO_INCREMENT,
 name VARCHAR(255) NOT NULL,
 manager_id BIGINT,
 PRIMARY KEY (employee_id),
 FOREIGN KEY (manager_id) REFERENCES employees(employee_id)
);
-- 사이즈 테이블 (CROSS JOIN 실습용)
CREATE TABLE sizes (
 size VARCHAR(10) PRIMARY KEY
);
-- 색상 테이블 (CROSS JOIN 실습용)
CREATE TABLE colors (
 color VARCHAR(20) PRIMARY KEY
);
```

```sql
-- 고객 데이터 입력
INSERT INTO users(name, email, address, birth_date) VALUES
('션', 'sean@example.com', '서울시 강남구', '1990-01-15'),
('네이트', 'nate@example.com', '경기도 성남시', '1988-05-22'),
('세종대왕', 'sejong@example.com', '서울시 종로구', '1397-05-15'),
('이순신', 'sunsin@example.com', '전라남도 여수시', '1545-04-28'),
('마리 퀴리', 'marie@example.com', '서울시 강남구', '1867-11-07'),
('레오나르도 다빈치', 'vinci@example.com', '이탈리아 피렌체', '1452-04-15');
-- 상품 데이터 입력
INSERT INTO products(name, category, price, stock_quantity) VALUES
('프리미엄 게이밍 마우스', '전자기기', 75000, 50),
('기계식 키보드', '전자기기', 120000, 30),
('4K UHD 모니터', '전자기기', 350000, 20),
('관계형 데이터베이스 입문', '도서', 28000, 100),
('고급 가죽 지갑', '패션', 150000, 15),
('스마트 워치', '전자기기', 280000, 40);
-- 주문 데이터 입력
INSERT INTO orders(user_id, product_id, quantity, status, order_date) VALUES
(1, 1, 1, 'COMPLETED', '2025-06-10 10:00:00'),
(1, 4, 2, 'COMPLETED', '2025-06-10 10:05:00'),
(2, 2, 1, 'SHIPPED', '2025-06-11 14:20:00'),
(3, 4, 1, 'COMPLETED', '2025-06-12 09:00:00'),
(4, 3, 1, 'PENDING', '2025-06-15 11:30:00'),
(5, 1, 1, 'COMPLETED', '2025-06-16 18:00:00'),
(2, 1, 2, 'SHIPPED', '2025-06-17 12:00:00');
-- 직원 데이터 입력
INSERT INTO employees(employee_id, name, manager_id) VALUES
(1, '김회장', NULL),
(2, '박사장', 1),
(3, '이부장', 2),
(4, '최과장', 3),
(5, '정대리', 4),
(6, '홍사원', 4);
-- 사이즈 데이터 입력
INSERT INTO sizes(size) VALUES
('S'), ('M'), ('L'), ('XL');
-- 색상 데이터 입력
INSERT INTO colors(color) VALUES
('Red'), ('Blue'), ('Black');
```

---

## 조인이 필요한 이유
대표님이 최근 주문 현황을 고객 이름과 상품명을 포함해서 보고서로 만들어줘라고 요청
- 주문 현황이 필요하기 때문에 orders의 정보를 제공하면 될 것 같음

### 문제점
- 고객 이름과 상품명은 orders에 없음
    - user_id, product_id만 가지고 있음
    - users를 통해서 고객 이름을 찾아야함
    - products를 통해서 상품 이름을 찾아야함

- 그렇다면 처음부터 하나의 테이블에 다 적으면 되는 것 아닐까?

### 만약 모든 데이터를 하나의 테이블에 저장한다면? 문제 발생
- 데이터 중복(Redundancy)
    - 특정 고객이 상품을 100번 주문
    - 이름, 이메일, 주소 정보가 100번이나 불필요하게 반복 저장

- 갱신 이상(Update Anomaly)
    - 특정 고객이 이메일을 변경
    - 특정 고객을 찾아서 전부 다 변경을 해주어야함
    - 만약, 실수로 단 하나라도 누락된다면 일관성이 깨져버림(어떤 정보가 진짜인지 알 수 없음)

- 삽입 이상(Insertion Anomaly)
    - 주문이 발생해야만 상품을 등록할 수 있음

- 삭제 이상(Deletion Anomaly)
    - 특정 고객이 딱 한 번만 주문한 기록이 있다면
    - 주문 데이터(현재 하나의 테이블)를 삭제하는 순간 고객 정보가 날아가버림

> 이러한 문제들 때문에 설계할 때 **정규화**라는 과정을 거침. 중복을 최소화하고, 이상 현상 방지

### 그래서 조인이 필요함
- 데이터를 분리해서 저장함
- 그래서 의미 있는 정보를 다시 얻으려면, **합쳐야함**

### 조인
- 조인은 두 개 이상의 테이블을 특정 컬럼을 기준으로 연결하여, 마치 처음부터 하나의 테이블이었던 것처럼 보여주는 기능
- 보통 (기본키 / 외래키)를 활용하여 이들을 합침
    - orders 테이블의 user_id 외래키는 users 테이블의 user_id 기본키와 연결
    - orders 테이블의 product_id 외래키는 products 테이블의 product_id 기본키와 연결

- **내부 조인 / 외부 조인**으로 나눌 수 있음

---

## 내부 조인 1

### 예제 요청 1
- 주문이 완료된(COMPLETED) 모든 주문에 대해, 어떤 고객이 주문했는지 고객 ID, 고객 이름과 주문 날짜를 함께 보고 싶은 경우 

### 내부 조인의 개념
- 두 테이블을 연결할 때, 양쪽 테이블에 모두 공통으로 존재하는 데이터만 결과로 보여줌
- 기준이 되는 컬럼(orders.user_id == users.user_id)의 값이 서로 일치하는 행들만 짝을 지어주는 것
    - orders 테이블에 user_id가 1인 주문이 있고, users 테이블에 user_id가 1인 회원이 있다면 둘은 함께 결과에 포함
    - orders 테이블에 user_id가 99인 주문은 있지만, users 테이블에 user_id가 99인 회원이 없다면, 결과에서 제외됨 (실제로는 데이터 무결성을 위해 외래 키 제약조건을 사용하므로 이런 경우는 발생하지 않음)

### 내부 조인 문법
```sql
select 컬럼1, 컬럼2, ...
from 테이블A
inner join 테이블B
    on 테이블A.연결컬럼 = 테이블B.연결컬럼;
```
- on절: 두 테이블을 어떤 조건으로 연결할지 명시하는 연결고리
    - 참이 되는 행들만 결과에 포함됨

> INNER 생략: 내부 조인에서 INNER는 생략 가능

### 실습
```sql
select *
from orders
inner join users
	on orders.user_id = users.user_id;
```

### Join의 과정
- join on에 의해서 orders와 users를 user_id를 기준으로 조인함
    - order_id: 1의 경우, user_id: 1 => users의 user_id: 1 항목과 연결
    - ... 
    - 연결할 대상이 없으면 내부 조인 대상에 포함되지 않음

### 예제 요청 1에 대한 쿼리
```sql
select 
    users.user_id,      -- 테이블 명 생략 가능
    users.name,         -- 테이블 명 생략 가능
    orders.order_date   -- 테이블 명 생략 가능
from orders
inner join users
	on orders.user_id = users.user_id
where orders.status = 'COMPLETED'; -- status라고 해도 됨(테이블 명을 적어주는 것이 좋음!!)
```

### 논리적인 순서
- from/join: from절의 orders 테이블과 inner join으로 연결된 users 테이블을 연결하기 위해 on절에 명시된 orders.user_id = users.user_id 조건을 만족하는 행들을 결합하여 하나의 큰 가상 테이블을 생성함
    - orders 영역 | users 영역

- where: join을 통해 생성된 가상 테이블에서 where 절의 조건인 order.status = "COMPLETED"를 만족하는 행들만 필터링함

- select: select를 통해서 원하는 열을 가져옴

### 순서 정리
- 논리적인 순서
    - FROM/JOIN(테이블 결합) -> where(조건 필터링) -> SELECT(컬럼 선택)

### 쿼리 최적화하기(쿼리 옵티마이저)
- 사용자와 데이터베이스 간의 논리적 순서는 일종의 약속
- 데이터베이스 내부의 쿼리 최적화기는 쿼리를 더 효율적인 방식으로 실행함
- 예를 들어서, orders 테이블에서 COMPLETED 상태의 행만 먼저 선택한 다음, 남은 행을 기준으로 users 테이블과 조인하는 방식으로 진행할수도 있음

- 즉, 물리적인 실행 순서는 달라질 수 있지만, 최종 결과는 논리적인 순서와 동일함
- 데이터베이스의 최적화 방식을 잘 이해하면 같은 결과를 얻으면서도 조회 성능을 최적화할 수 있음

---

## 내부 조인 2

### 조인과 집합
- 내부 조인을 이해하는 또 다른 방법은 **집합**의 관점에서 바라보는 것
- 내부 조인은 두 테이블의 **교집합**을 찾는 것과 같음
    - A집합: orders 테이블에 있는 모든 사용자들의 user_id 집합
    - B집합: users 테이블에 있는 모든 user_id 집합
```sql
select distinct user_id from orders; -- 1 2 3 4 5(중복 값 제거)
select user_id from users; -- 1 2 3 4 5 6
```
- 내부 조인의 의미는 벤다이어그램에서 겹쳐지는 영역을 말함
    - 외부 조인은 교집합 영역의 밖(OUTER)의 행까지 포함한다는 의미

### 내부 조인과 조인 방향
- 내부 조인은 양방향
- A->B로 조인할 수 있다면 반대로 B->A로 조인할 수 있고, 그 결과는 항상 같음!!
```sql
select 
	orders.order_id,
	orders.order_date,
	orders.user_id as orders_user_id,
	users.user_id as users_user_id,
	users.name
from orders -- A->
inner join users -- B
	on orders.user_id = users.user_id 
order by order_id

select 
	orders.order_id,
	orders.order_date,
	orders.user_id as orders_user_id,
	users.user_id as users_user_id,
	users.name
from users -- B->
inner join orders -- A
	on orders.user_id = users.user_id 
order by order_id
```
- 결과: 완전히 똑같음

---

## 내부 조인 3

### 실무 팁: 조인 순서는 언제 중요할까?
- 내부 조인에서는 결과가 같으므로 어떤 순서로 작성해도 무방함
- 쿼리를 읽는 사람의 입장에서 **어떤 데이터가 중심이 되는가에 따라 순서를 정하면 가독성이 높아짐**
    - 주문 목록을 중심으로 고객 정보를 추가하고 싶다면 from orders join users
    - 고객 목록을 중심으로 주문 정보를 추가하고 싶다면 from users join orders

> 참고: 외부 조인은 조인 순서가 결과에 매우 큰 영향을 미침
- 질문: 한 번도 주문하지 않은 고객은 어떻게 찾을 수 있을까?
    - 내부 조인으로 해결할 수 없음
    - 외부 조인을 사용해야함

### 가독성을 높이는 테이블 별칭
- as를 사용하거나
- 한 칸 띄우고 원하는 별칭을 붙이면 됨
```sql
select 
    o.order_id,
    o.order_date,
    o.user_id as orders_user_id, -- 컬럼에 사용하는 별칭은 생략하지 않음
    u.user_id as users_user_id,
    u.name
from orders o -- 테이블에 사용하는 별칭은 생략하고
join users u -- inner 생략
    on o.user_id = u.user_id
```

### 실무 팁: 테이블과 컬럼의 별칭 AS 생략
- 테이블 별칭: AS 생략
    - 테이블 이름 뒤에 오는 별칭은 문법적으로 혼동의 여지가 거의 없음

- 컬럼 별칭: AS 사용
    - 가독성이 낮은 예시
    ```sql
    select salary * 12 annual_salary
    ```

    - 가독성이 높은 예시
    ```sql
    select salary * 12 as annual_salary
    ```

--- 

## 문제와 풀이

### 주문별 상품 정보 조회
```sql
select
    o.order_id,
    p.name,
    o.quantity
from
    orders o
join
    products p on o.product_id = p.product_id
order by
 o.order_id;
```

### 3개 테이블 조인하기
```sql
select 
	o.order_id,
	u.name,
	p.name,
	o.order_date
from
	orders o
join
	users u on o.user_id = u.user_id
join 
	products p on o.product_id = p.product_id 
where
	o.status = 'SHIPPED';
```

### 고객별 총 구매액 계산
```sql
select
    u.name as user_name,
    sum(o.quantity * p.price ) as total_purchase_amount
from
    orders o
join users u
    on o.user_id = u.user_id
join products p
	on o.product_id = p.product_id
group by
    u.name
order by
	total_purchase_amount desc;
```
