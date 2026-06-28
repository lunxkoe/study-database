# S10. 데이터 무결성

## 데이터 무결성이 중요한 이유

### 문제 상황
- 총 매출액 마이너스
- 존재하지 않는 99번 고개의 주문 기록 잔여
> **쓰레기 데이터**

### 쓰레기 데이터가 초래하는 재앙
- 잘못된 비즈니스 결정: 매출액이 음수로 찍힌 보고서를 믿고 다음 달 사업 계획을 세울 수는 없음
- 치명적인 시스템 오류: 항상 양수일 것이라고 가정하고 만들었는데, 음수 가격이 들어오면 안됨
- 데이터 불일치: 탈퇴한 고객을 users에서는 삭제했는데, ordres에는 그 고객의 주문 기록이 그대로 남아 "주인 없는 주문"이라는 유령 데이터가 떠돌아다님

### 책임은 누구에게 있을까?
- 애플리케이션에 미처 발견하지 못한 버그가 있을 수 있음
- 여러 다른 종류의 애플리케이션이 하나의 데이터베이스에 접근할 수도 있음
- 개발자나 관리자가 데이터베이스에 직접 접속해서 데이터를 수정할 수도 있음

> 결론: 데이터베이스를 데이터를 지키는 최후의 보루로 만들어야만 함!!
- 이상한 데이터가 생기거나 들어오면 데이터베이스 스스로가 거부할 수 있게 만들어야함

### 제약 조건(Constraint)의 역할
- INSERT, UPDATE, DELETE를 할 때, 이 규칙을 어기면 절대 안돼라고 하는 문지기
    - price 컬럼에는 0이상의 값만 받도록 규칙을 정하고, 음수 값을 넣으려는 시도는 거부
    - email 컬럼에는 중복된 값이 들어올 수 없도록 규칙을 정하고, 이미 가입된 이메일은 거부
    - orders 테이블의 user_id는 반드시 users 테이블에 존재하는 user_id값만 받도록 규칙을 정하고, 유령 회원의 주문은 거부함

### 사용해본 제약 조건
- primary key
- foreign key
- not null

---

## 기본 제약 조건

### NOT NULL: null 값 방지
- 역할: 해당 컬럼에 null값이 저장되는 것을 허용하지 않음. 반드시 필요한 정보가 누락되는 막음
- 문법: email varchar(255) not null

### 위반 시나리오
```sql
insert into users (name, email) values('냐옹이', null);
-- SQL Error [1048] [23000]: Column 'email' cannot be null
```

### UNIQUE
- 역할: 해당 컬럼에 들어가는 모든 값은 테이블 내에서 반드시 고유해야함. 중복 데이터가 쌓이는 것을 막음
- 문법: email varchar(255) unique

### 위반 시나리오
```sql
insert into users (name, email, address) values('가짜 션', 'sean@example.com', '서울시 어딘가');
-- SQL Error [1062] [23000]: Duplicate entry 'sean@example.com' for key 'users.email'
```

### PRIMARY KEY: 행의 대표 식별자
- 역할: 테이블의 각 행을 고유하게 식별할 수 있는 단 하나의 대표키: NOT NULL / UNIQUE 제약 조건의 특징을 모두 포함. 기본 키 컬럼은 절대 null일수도 없으며, 절대 중복될 수 없음
- 특징: 테이블 당 오직 하나만 설정 가능. 성능에도 매우 중요한 역할을 함
    - MySQL은 기본 키에 자동으로 고성능 인덱스를 생성함
- 문법: user_id BIGINT PRIMARY KEY

### 위반 시나리오
```sql
insert into users(user_id, name, email) value (1, '누군가', 'test@email.com');
-- SQL Error [1062] [23000]: Duplicate entry '1' for key 'users.PRIMARY'
```
- 참고: 현재 user_id는 auto_increment를 사용해서 null을 입력해도 됨. 안했는데 null 입력 시 오류 발생

### Default: 기본값 설정
- 역할: 특정 컬럼에 값을 명시적으로 입력하지 않은 경우, 자동으로 설정된 기본값을 저장
    - 무결성을 강제하는 것. 제약조건까지는 아님

### 활용 시나리오
```sql
insert into orders(user_id, product_id, quantity) values (2, 2, 1);
select * from orders order by order_id desc;
-- status 컬럼에 pending으로 자동으로 들어감
```

### 또 다른 핵심: 관계
- 지금까지는 하나의 테이블 내부에서 데이터의 유효성을 지키는 구조
- **진짜 핵심은 바로 관계**

---

## 외래 키 제약 조건

### 관계의 중요성: 참조 무결성
- **두 테이블의 관계가 항상 유효해야함**
- orders의 user_id가 users에 없으면 관계가 깨진 것

### 외래 키의 역할: 유령 데이터를 막아라
- 자식 테이블(orders)
    - insert, update할 때: 부모 테이블(users)에 존재하지 않는 user_id 값을 자식 테이블(orders)의 user_id 컬럼에 넣으려는 시도를 막음(유령 주문 생성 방지)

- 부모 테이블(users)
    - delete, update를 할 때: 자식 테이블(orders)에서 참조하고 있는 user_id 값을 가진 행을 함부로 삭제하거나 변경하지 못하게 막음(기존 주문을 유령 주문으로 만드는 것을 방지)

### 위반 시나리오
- 유령 주문 생성 시도
```sql
insert into orders(user_id, product_id, quantity) values(999, 1, 2);
-- SQL Error [1452] [23000]: Cannot add or update a child row: a foreign key constraint fails (`my_shop2`.`orders`, CONSTRAINT `fk_orders_users` FOREIGN KEY (`user_id`) REFERENCES `users` (`user_id`))
```

- 주문이 있는 고객 삭제 시도
    - 주문이 없으면 삭제를 할 수 있음!!
```sql
delete from users where user_id = 1;
-- SQL Error [1451] [23000]: Cannot delete or update a parent row: a foreign key constraint fails (`my_shop2`.`orders`, CONSTRAINT `fk_orders_users` FOREIGN KEY (`user_id`) REFERENCES `users` (`user_id`))
```

### ON DELETE / ON UPDATE 옵션
- 부모 데이터의 삭제가 수정을 무조건 막는 것이 기본값(RESTRICT)이며 가장 안전한 정책
- 새로운 비즈니스 규칙:
    - 회원이 탈퇴하면, 그 회원의 모든 주문 기록도 함께 삭제되어야함

- 옵션
    - restrict(기본값): 자식 테이블에 참조하는 행이 있으면 부모 테이블의 행을 삭제/수정할 수 없음
    - cascade: 부모 테이블의 행이 삭제/수정되면, 그를 참조하는 자식 테이블의 행도 함께 자동으로 삭제/수정됨
    - set null: 부모 테이블의 행이 삭제/수정되면, 자식 테이블의 해당 외래 키 컬럼의 값을 NULL로 설정(단, 자식 테이블의 외래 키 컬럼이 NULL을 허용해야 함)

### ON DELETE CASCADE 실습
```sql
-- 실습을 위해 기존 테이블 삭제 후 CASCADE 옵션으로 재생성
DROP TABLE orders;
CREATE TABLE orders (
 order_id BIGINT AUTO_INCREMENT,
 user_id BIGINT NOT NULL,
 product_id BIGINT NOT NULL,
 order_date DATETIME DEFAULT CURRENT_TIMESTAMP,
 quantity INT NOT NULL,
 status VARCHAR(50) DEFAULT 'PENDING',
 PRIMARY KEY (order_id),
 CONSTRAINT fk_orders_users FOREIGN KEY (user_id)
 REFERENCES users(user_id) ON DELETE CASCADE, -- CASCADE 옵션 추가
 CONSTRAINT fk_orders_products FOREIGN KEY (product_id)
 REFERENCES products(product_id)
);

-- 션 회원 다시 등록
INSERT INTO users(user_id, name, email, address, birth_date) VALUES
(1, '션', 'sean@example.com', '서울시 강남구', '1990-01-15');

-- 주문 데이터 다시 입력
INSERT INTO orders(user_id, product_id, quantity, status) VALUES
(1, 1, 1, 'COMPLETED'),
(1, 4, 2, 'COMPLETED'),
(2, 2, 1, 'SHIPPED');
```

```sql
delete from users where user_id = 2;
```
- 이제는 삭제됨

### CASCADE 실무 이야기
- 편리함
- 잘못 사용하면 대량의 데이터가 삭제될 수 있음
- **잘 사용하지 않음**
- 애플리케이션 계층에서 명시적으로 관련된 데이터를 처리하는 방식이 더 좋음
    - 직접 삭제하는 로직을 사용

---

## CHECK 제약 조건

### 아직까지 해결하지 못한 문제
- 데이터의 내용 자체에 대한 규칙을 어떻게 적용할까?
    - 상품 가격이나 재고 수량은 절대 음수일 수 없음
    - 할인은 0%에서 100% 사이의 값이어야함

### CHECK 제약 조건의 역할과 문법
- 특정 컬럼에 대해 INSERT, UPDATE가 일어날 때마다 지정된 조건이 참인지를 검사하고,
- 거짓이면 데이터베이스는 해당 데이터의 입력을 거부하고 에러를 방지시킴

### 실습 준비
```sql
-- 실습을 위해 기존 테이블들을 삭제한다.
DROP TABLE IF EXISTS orders;
DROP TABLE IF EXISTS products;

-- CHECK 제약 조건을 추가하여 products 테이블 재생성
CREATE TABLE products (
 product_id BIGINT AUTO_INCREMENT PRIMARY KEY,
 name VARCHAR(255) NOT NULL,
 category VARCHAR(100),
 price INT NOT NULL CHECK (price >= 0),
 stock_quantity INT NOT NULL CHECK (stock_quantity >= 0),
 discount_rate DECIMAL(5, 2) DEFAULT 0.00 CHECK (discount_rate BETWEEN 0.00
AND 100.00)
);
```

### 위반 시나리오
- 가격에 음수값을 넣으려고 시도
```sql
INSERT INTO products (name, category, price, stock_quantity)
VALUES ('오류상품', '전자기기', -5000, 10);
-- SQL Error [3819] [HY000]: Check constraint 'products_chk_1' is violated.
```

- 할인율에 범위를 벗어난 값 입력 시도
```sql
INSERT INTO products (name, category, price, stock_quantity, discount_rate)
VALUES ('초특가상품', '패션', 50000, 20, 120.00);
-- SQL Error [3819] [HY000]: Check constraint 'products_chk_3' is violated.
```

### 행위에 대한 규칙
- 행위1: orders 테이블에 새로운 주문 정보를 기록
- 행위2: products 테이블에서 해당 상품의 재고를 줄임
> 이 두가지 행위는 절대로 쪼개져서는 안됨
- 둘 다 성공적으로 끝나거나, 하나라도 실패하면 둘 다 없었던 일로 되돌려야함!!!
> All or Nothing: 트랜잭션

### 데이터 검증 어디서 하는게 좋을까?
- 요즘 대세는 애플리케이션 코드: 대부분의 비즈니스 로직과 유효성 검사는 애플리케이션에서 직접 처리
- DB CHECK 제약 조건은 신중하게
- 최후의 방어선으로 활용: CHECK 제약 조건을 쓴다면, 간단하지만 절대 값이 바뀌면 안되는 핵심 데이터에만 최후의 방어선으로 사용하는 것이 좋음

- CHECK를 사용하지 않는 경우
    - 비즈니스 로직 변경이 일어남
    - DB / 애플리케이션 코드도 다 바꿔야할 수도 있음
        - 복잡한 건: 애플리케이션 코드
        - 핵심적인 데이터: CHECK 제약조건 

---

## 정리

### 기본 제약 조건
- NOT NULL
- UNIQUE
- PRIMARY KEY
- DEFAULT

### 외래 키 제약 조건
- 삭제/수정 함부로 안됨
- ON DELETE / ON UPDATE
    - RESTRICT
    - CASCADE
    - SET NULL
- CASCADE보다는 애플리케이션에서 연쇄 삭제 처리를 하는 것이 좋음

### CHECK 제약 조건
- CHECK
- INSERT / UPDATE 시 사용
- 최후의 보루

### 외래 키 제약조건
- 외래 키 제약조건을 기본적으로 사용
- 너무 서비스가 커지면 풀고 해도 되긴함
- 일단 사용하자!!
    - 데이터 무결성이 깨지는 경우가 많이 발생함

### 추가 - UPDATE ON CASCADE
- 수정을 막는 범위는 외래 키로 사용하는 컬럼이 바뀌는 것을 막는 것
- 그 외 다른 부모의 컬럼은 변경되도 됨
- 수정되면 자식의 외래키가 같이 수정됨
