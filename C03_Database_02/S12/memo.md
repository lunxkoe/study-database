# S12. 저장 프로시저, 함수, 트리거

## 프로시저, 함수, 트리거 - 소개

### 문제 상황
- 쇼핑몰에서 새로운 회원 가입할 때, 시스템은 다음과 같은 여러 작업을 순서대로 처리해야함
    - users 테이블에 새로운 고객 정보를 insert
    - user_profiles 테이블에 고객의 상세 프로필 정보를 insert
    - couponse 테이블에 신규 가입 축하 쿠폰을 insert
    - logs 테이블에 신규 회원이 가입했다는 기록을 insert

- 위 4개의 SQL문을 묶음으로 회원가입이라는 로직을 작성
    - 애플리케이션에서 작성할 수도 있지만
    - 데이터베이스 안에 저장해두고 회원가입_처리('네이트', ...)를 호출하도록 할 수 없을까?

- 저장 프로시저, 저장 함수, 트리거를 제공함

### 데이터베이스 속 작은 프로그램들
1. 저장 프로시저
    - 이름이 부여된 일련의 SQL 작업 묶음
    - 파라미터를 받아 로직 처리 가능
        - 제어문 사용도 가능
        - 여러 개의 insert, update, delete작업을 포함하는 복잡한 비즈니스 로직을 하나의 단위로 처리하는데 사용
    - CALL 프로시저이름(파라미터1, 파라미터2, ...)와 같이 CALL 명령어로 독립적으로 호출

2. 저장 함수
    - 특정 계산을 수행하고 **반드시 하나의 값을 반환하는 프로그램**
    - 프로시저와 달리 반드시 하나의 값을 반환(Return)해야함
        - SUM(), COUNT() 같이 내장 함수처럼 일반적인 select 쿼리문 안에서 값의 일부로 사용될 수 있음
    - SELECT name, 나의함수(컬럼명) from 테이블;

3. 트리거
    - 특정 테이블에 특정 이벤트가 발생했을 때, 자동으로 실행되도록 약속된 프로그램
    - 개발자가 직접 호출하는 것이 아니라, 특정 조건이 만족되면 데이터베이스에 의해 자동으로 실행됨
        - 이벤트: insert, update, delete같은 데이터 변경 작업을 의미
    - orders테이블에 새로운 주문이 들어올 때마다, 자동으로 배송 테이블에 새로운 배송 준비 데이터를 생성하거나, logs 테이블에 감사 기록을 남기는 용도로 사용될 수 있음

### 왜 사용할까?
- 성능: 데이터베이스 안에 로직을 넣어두고 사용하면 네트워크 비용이 한 번만 사용할 수 있음
- 코드 재사용과 중앙화
- 보안: 특정 프로시저만 실행할 수 있는 권한만 주면 됨

---

## 프로시저, 함수, 트리거 - 실습
> 잘 사용하지 않거나 아예 사용하지 않음

### 저장 프로시저 실습
```sql
DROP TABLE IF EXISTS logs;
CREATE TABLE logs (
 id INT AUTO_INCREMENT PRIMARY KEY,
 description VARCHAR(255),
 created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

- 프로시저 생성
    - MySQL 클라이언트에서 여러 줄의 명령어로 이루어진 프로시저를 만들 때는, 명령어의 끝을 알리는 구분자(delimiter)를 세미콜론( ; )이 아닌 다른 기호(예: // 또는 $$ )로 잠시 변경
```sql
-- 구분자를 // 로 변경
DELIMITER //
CREATE PROCEDURE sp_change_user_address(
 IN user_id_param INT,
 IN new_address_param VARCHAR(255)
)
BEGIN
 -- 1. users 테이블의 주소 업데이트
 UPDATE users SET address = new_address_param WHERE user_id =
user_id_param;

 -- 2. logs 테이블에 변경 이력 기록
 INSERT INTO logs (description)
 VALUES (CONCAT('User ID ', user_id_param, ' 주소 변경 ', new_address_param));
END //
-- 구분자를 다시 ; 로 원상 복구
DELIMITER ;

select * from users
where user_id = 2; 
call sp_change_user_address(2, '경기도 성남시2');

-- 프로시저 삭제
DROP PROCEDURE IF EXISTS sp_change_user_address;
```

## 저장 함수 실습
```sql
DROP TABLE IF EXISTS stored_items;
CREATE TABLE stored_items (
 item_id BIGINT AUTO_INCREMENT PRIMARY KEY,
 name VARCHAR(255) NOT NULL,
 price INT NOT NULL,
 discount_rate DECIMAL(5, 2)
);
INSERT INTO stored_items (name, price, discount_rate) VALUES
('고성능 노트북', 1500000, 10.00),
('무선 마우스', 25000, 20.00),
('기계식 키보드', 120000, 30.00),
('4K 모니터', 450000, 40.00),
('전동 높이조절 책상', 800000, 50.00);
```

- 함수 생성
    - 함수는 프로시저와 달리 반드시 RETURNS로 반환 타입을 명시해야함
```sql
DELIMITER //
CREATE FUNCTION fn_get_final_price(
 price_param INT,
 discount_rate_param DECIMAL(5, 2)
)
RETURNS DECIMAL(10, 2)
DETERMINISTIC
BEGIN
-- 최종 가격을 계산 (가격 * (1 - 할인율/100))
 RETURN price_param * (1 - discount_rate_param / 100);
END //
DELIMITER ;
```

```sql
SELECT
 name,
 price,
 discount_rate,
 fn_get_final_price(price, discount_rate) AS final_price
FROM stored_items;

-- 함수 삭제
DROP FUNCTION IF EXISTS fn_get_final_price;
```

### 트리거 실습
```sql
-- 본 실습을 위한 탈퇴 고객 테이블 생성
DROP TABLE IF EXISTS retired_users;
CREATE TABLE retired_users (
 id BIGINT PRIMARY KEY,
 name VARCHAR(255) NOT NULL,
 email VARCHAR(255) NOT NULL,
 retired_date DATE NOT NULL
);
-- 탈퇴 고객 데이터 입력
INSERT INTO retired_users (id, name, email, retired_date) VALUES
(1, '션', 'sean@example.com', '2024-12-31'),
(7, '아이작 뉴턴', 'newton@example.com', '2025-01-10');
```

- 트리서 생성
    - BEFORE 또는 AFTER 특정 이벤트(insert, update, delete)에 대해 정의함
    - OLD는 변경 전 데이터, NEW는 변경 후 데이터를 의미
```sql
DELIMITER //
CREATE TRIGGER trg_backup_user
BEFORE DELETE ON users
FOR EACH ROW
BEGIN
 INSERT INTO retired_users (id, name, email, retired_date)
 VALUES (OLD.user_id, OLD.name, OLD.email, CURDATE());
END //
DELIMITER ;
```

```sql
DELETE FROM users WHERE user_id = 5;

SELECT * FROM users WHERE user_id = 5;

SELECT * FROM retired_users WHERE id = 5;

--- 트리거 삭제
DROP TRIGGER IF EXISTS trg_backup_user;
```

## 데이터베이스 로직의 함정과 현대적 대안

### 사용하지 않는 이유
1. 유지보수의 어려움
    - 데이터베이스에 비즈니스 로직이 들어가기 시작하면, 전체 시스템의 로직이 애플리케이션 코드와 데이터베이스 양쪽에 흩어지게 됨

2. 성능 및 확장성 문제
    - 과거에는 DB가 애플리케이션 서버보다 월등히 성능이 좋았음
    - 지금은 상황이 바뀜

3. 특정 데이터베이스에 대한 종속성(한 번 종속되면 빠져나올 수 없음)
    - DBMS마다 저장 프로시저, 함수, 트리거를 작성하는데 사용되는 절차적 SQL언어는 데이터베이스 제조사마다 다름

### 현대의 대안: 명확한 역할 분리
- 애플리케이션 계층: 
    - 비즈니스 로직, 데이터 가공, 조건 처리, 절차 제어 등 모든 지능적인 처리를 담담
    - 데이터베이스와는 표준 SQL을 통해 소통함

- 데이터베이스 계층:
    - 오직 데이터의 저장, 조회, 무결성 보장(제약 조건), 트랜잭션 관리라는 데이터 본연의 역할에만 충실

### 최종 정리
- 완전히 쓸모 없는 것은 아님
- 데이터 웨어하우징 환경에서는 대용량 데이터를 배치 처리하거나, DBA가 특정 관리 작업을 자동화하는 등 여전히 유용하게 쓰이는 특정 영역이 존재함
