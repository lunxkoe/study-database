# S12. 물리적 모델링 실습

## 인덱스 설계 - 실습

### MySQL 인덱스 자동 생성
- Primary Key / UNIQUE Key / Foreign Key를 설정하면 자동으로 해당 컬럼에 인덱스가 생성됨

### 시나리오 3: 상품명으로 상품 검색
- 사용자가 쇼핑몰 상단의 검색창에 노트북이라고 입력하여 상품을 검색
- product_name에 **like**를 사용 (인덱스 필요)

- **주의**
    - like %검색어%처럼 검색어 앞에 %가 붙는 경우에는 인덱스가 제대로 동작하지 않음
    - 이런 전문적인 텍스트 검색 기능을 위해서는 MySQL의 Full-Text Index나 Elasticsearch 같은 별도의 검색 엔진을 사용하는 것이 실무적인 해결책
    - 단, 앞에 %가 없는 경우는 잘 동작함

### 시나리오 4: 관리자의 주문 상태 및 기간별 조회(복합 인덱스)
- orders -> order_status, ordered_at
- 복합 인덱스를 만들어주는 것이 훨씬 더 효율적

- **중요**
    - 복합 인덱스의 컬럼 순서
    - 첫 번째 컬럼을 기준으로 정렬되고, 두 번째 컬럼을 기준으로 정렬이 되는 방식으로 동작함
    - 선택도가 높고, = 조건으로 사용되는 컬럼 앞에 두는 것이 일반적인 원칙
    - order_status는 몇 가지 값으로 정해져 있어서 선택도는 낮음
        - 하지만 = 조건으로 활용함으로 무조건 앞에 두어야 함
    - ordered_at은 범위 조건으로 사용됨

- 복합 인덱스 대원칙
    - 인덱스는 순서대로 사용해라
    - 등호 조건은 앞으로, 범위 조건은 뒤로
    - 정렬도 인덱스 순서를 따르라

### 시나리오 5: 사용자의 주문 상태 및 기간별 조회
- member_id를 가지면서, order_status와 ordered_at 조건까지 만족하는 데이터를 찾음
- **이 경우에는 인덱스를 만들지 말지 고민이 필요함**

- member_id의 압도적으로 높은 선택도
    - 인덱스를 사용할 때는 기본적으로 선택도가 높아야 함
    - 선택도가 높다는 것은 데이터가 많이 걸러진다는 의미(반대: 성별, 두 가지 밖에 없음)
    
- 인덱스 사용 후의 동작 과정
    - idx_member_id 인덱스를 사용해서 10만 주문 데이터를 빠르게 찾음
    - 이 데이터를 메모리로 가져옴
    - 여기서 이 소량의 데이터를 대상으로 order_status, ordered_at을 필터링함
    > 수백만건 => 수백건으로 줄어들어 무시할 수 있을 정도로 빠름

### 인덱스의 비용: 모든 쿼리에 인덱스를 만들지 않는 이유
- (member_id, order_status, ordered_at) 순서로 복합 인덱스를 만든다면 조금 더 빨라질 수 있는 것은 당연
- 트레이드 오프
    - 저장 공간 비용
    - 쓰기 성능 저하
        - 조회 속도가 증가되긴 하지만, 쓰기 속도가 줄어듦

- 주문은 쓰기가 많이 일어남
    - 인덱스 생성에 부담이 좀 있음

### 인덱스 추가를 결정하는 실무적 기준
- 데이터 분포와 조회 효율성
    - 선택도(member_id)
    - 한 회원의 주문이 수십 ~ 수백 건 수준이라면 member_id 인덱스만으로 충분히 빠름

- 쓰기 작업의 빈도와 중요도

- 실제 쿼리 패턴과 성능 측정(가장 중요)
    - EXPLAIN 실행 계획 분석:
        - 실제 쿼리가 member_id 인덱스를 사용한 후 얼마나 많은 행을 스캔하는지
        - 여기서 스캔하는 행의 수가 많다고 판단되면 인덱스 추가를 검토
    - 성능 테스트:
        - 개발 환경에서 인덱스와 데이터를 추가하고, 추가하기 전과 후의 조회 성능 및 주문 생성/수정 성능을 직접 비교 측정

---

## 역정규화 - 실습

### 주문 조회 시 상품명 조회
- product_name을 추가

- 실무 경험 기반 설명
    - 데이터 불일치 문제가 발생할 수도 있음
    - 사실 이건 역정규화가 아닐 수도 있음
    - 과거 주문 내역의 기록 자체로 관리할 수도 있게 됨(역사적 데이터)

### 계산된 값 저장(파생 컬럼 추가)
- 조회 성능 증가
- 단, 쓰기 작업의 성능 및 **복잡성 증가**

### 테이블 합치기 (orders - delivery)
- 결론: 왠만하면 하지 말자
    - 확장성이 많이 깨짐
    - 응집도가 깨짐

- 생명주기가 다른 데이터는 분리하자
    - orders - delivery

- 변경 빈도가 다른 데이터는 분리하라
    - 주문의 핵심 정보는 한 번 정해지면 바뀔 일이 거의 없는 정적인 데이터
    - 배송 상태는 계속해서 update가 일어나는 동적인 데이터
    - 고작 배송 상태를 바꾸기 위해서 주문 정보까지 함께 조회하는 것은 불필요함

---

## 쇼핑몰 테이블 정의서

### 정의서
- 용어 사전집 기반으로 작성

- 항목 1
    - 테이블 한글명
    - 테이블 영문명
    - 테이블 설명

- 항목 2 (복합 인덱스 표현)
    - No.
    - 컬럼 한글명
    - 컬럼 영문명
    - 데이터 타입
    - 제약 조건
    - 비고(기능 설명)

- 항목 3
    - **별도의 인덱스**

### 생성일과 수정일
- 실무에서는 모든 테이블에 생성일과 수정일을 넣는 것이 거의 표준처럼 여겨짐
- 실무 팁
    - 생성자 / 수정자
    - 이것도 넣어두는 것이 좋음
    - 관리자가 수정한 것인지, 사용자가 수정한 것인지 찾을 수 있으면 좋음
    - 애플리케이션에서 스프링 데이터 JPA를 활용하면 관리할 수 있음

### 주문일와 수정일(비즈니스 시간과 시스템 시간은 반드시 분리)
- 주문일은 비즈니스 시간
- 수정일은 시스템 시간

- 시스템 장애 또는 지연
    - 주문일은 주문이 생성될 때 기점으로 넣어주고
    - 수정일은 실제 시스템에 저장될 때 기점으로 넣어주고
    - 따라서 고객이 정확히 잘 주문했는지 확인할 수 있음

---

## 쇼핑몰 DDL

### 쇼핑몰 DB 구축 통합 스크립트 - SQL
```sql
------------------------------------------
-- 데이터베이스 초기화
------------------------------------------
DROP DATABASE IF EXISTS my_shop3_ex;
CREATE DATABASE my_shop3_ex;
USE my_shop3_ex;
------------------------------------------
-- 테이블 초기화 (외래 키 종속성을 고려하여 자식 테이블부터 삭제)
------------------------------------------
DROP TABLE IF EXISTS pay;
DROP TABLE IF EXISTS delivery;
DROP TABLE IF EXISTS order_item;
DROP TABLE IF EXISTS orders;
DROP TABLE IF EXISTS product;
DROP TABLE IF EXISTS member;
-- 1. member 테이블
CREATE TABLE member (
 member_id BIGINT NOT NULL AUTO_INCREMENT, -- 회원 ID (PK)
 login_id VARCHAR(50) NOT NULL, -- 로그인 ID
 password VARCHAR(255) NOT NULL, -- 비밀번호 (암호화)
 member_name VARCHAR(50) NOT NULL, -- 이름
 email VARCHAR(100) NOT NULL, -- 이메일
 addr VARCHAR(255) NULL, -- 주소
 created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
 updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE
CURRENT_TIMESTAMP,
 PRIMARY KEY (member_id),
 UNIQUE KEY uq_login_id (login_id),
 UNIQUE KEY uq_email (email)
);
-- 2. product 테이블
CREATE TABLE product (
 product_id BIGINT NOT NULL AUTO_INCREMENT, -- 상품 ID (PK)
 product_name VARCHAR(100) NOT NULL, -- 상품명
 product_price INT NOT NULL, -- 가격
 stock_quantity INT NOT NULL DEFAULT 0, -- 재고 수량
 created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
 updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE
CURRENT_TIMESTAMP,
 PRIMARY KEY (product_id),
 INDEX idx_product_name (product_name) -- 상품명 검색용 인덱스
);
-- 3. orders 테이블
CREATE TABLE orders (
 order_id BIGINT NOT NULL AUTO_INCREMENT, -- 주문 ID (PK)
 member_id BIGINT NOT NULL, -- 회원 ID (FK)
 ordered_at DATETIME NOT NULL, -- 주문일 (애플리케이션에서 생성)
 order_status VARCHAR(20) NOT NULL DEFAULT 'ORDERED', -- 주문 상태
 total_amount INT NOT NULL, -- 총 주문 금액 (역정규화)
 created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
 updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE
CURRENT_TIMESTAMP,
 PRIMARY KEY (order_id),
 CONSTRAINT fk_orders_member FOREIGN KEY (member_id)
 REFERENCES member (member_id),
 INDEX idx_order_status_ordered_at (order_status, ordered_at) -- 관리자용 주문 조
회 인덱스
);
-- 4. order_item 테이블
CREATE TABLE order_item (
 order_item_id BIGINT NOT NULL AUTO_INCREMENT, -- 주문 상품 ID (PK)
 order_id BIGINT NOT NULL, -- 주문 ID (FK)
 product_id BIGINT NOT NULL, -- 상품 ID (FK)
 product_name VARCHAR(100) NOT NULL, -- 주문 당시 상품명 (역정규화)
 order_price INT NOT NULL, -- 주문 당시 가격
 order_quantity INT NOT NULL, -- 주문 수량
 created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
 updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE
CURRENT_TIMESTAMP,
 PRIMARY KEY (order_item_id),
 CONSTRAINT fk_order_item_orders FOREIGN KEY (order_id)
 REFERENCES orders (order_id),
 CONSTRAINT fk_order_item_product FOREIGN KEY (product_id)
 REFERENCES product (product_id),
 -- [추가됨] 주문 ID와 상품 ID에 대한 유니크 제약 조건
 CONSTRAINT uq_order_id_product_id UNIQUE (order_id, product_id)
);
-- 5. delivery 테이블
CREATE TABLE delivery (
 delivery_id BIGINT NOT NULL AUTO_INCREMENT, -- 배송 ID (PK)
 order_id BIGINT NOT NULL, -- 주문 ID (FK, Unique)
 delivery_status VARCHAR(20) NOT NULL DEFAULT 'READY',-- 배송 상태
 tracking_no VARCHAR(50) NULL, -- 운송장 번호
 ship_addr VARCHAR(255) NOT NULL, -- 배송지
 created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
 updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE
CURRENT_TIMESTAMP,
 PRIMARY KEY (delivery_id),
 UNIQUE KEY uq_delivery_order_id (order_id), -- 주문 하나당 배송은 하나
 CONSTRAINT fk_delivery_orders FOREIGN KEY (order_id)
 REFERENCES orders (order_id)
);
-- 6. pay 테이블
CREATE TABLE pay (
 pay_id BIGINT NOT NULL AUTO_INCREMENT, -- 결제 ID (PK)
 order_id BIGINT NOT NULL, -- 주문 ID (FK, Unique)
 pay_method VARCHAR(50) NOT NULL, -- 결제 수단
 pay_amount INT NOT NULL, -- 결제 금액
 pay_status VARCHAR(20) NOT NULL, -- 결제 상태
 paid_at DATETIME NULL, -- 결제 완료일
 created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
 updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE
CURRENT_TIMESTAMP,
 PRIMARY KEY (pay_id),
 UNIQUE KEY uq_pay_order_id (order_id), -- 주문 하나당 결제는 하나
 CONSTRAINT fk_pay_orders FOREIGN KEY (order_id)
 REFERENCES orders (order_id)
);
```

### 쇼핑몰 DB 구축 통합 스크립트 - 샘플 DML
```sql
------------------------------------------
-- 샘플 데이터 입력 (DML)
------------------------------------------
-- 1. 회원 데이터
-- 실제 서비스에서는 비밀번호를 반드시 암호화(해싱)하여 저장해야 한다.
INSERT INTO member(login_id, password, member_name, email, addr)
VALUES
('sejong', 'pass123!', '세종대왕', 'sejong@example.com', '서울시 종로구'),
('sunsin', 'pass456!', '이순신', 'sunsin@example.com', '전라남도 여수시'),
('curie', 'pass789!', '마리 퀴리', 'curie@example.com', '프랑스 파리'),
('nate', 'pass101!', '네이트', 'nate@example.com', '서울시 송파구'),
('jobs', 'pass112!', '스티브 잡스', 'jobs@example.com', '미국 캘리포니아'),
('sean', 'pass131!', '션', 'sean@example.com', '성남시 분당구');
-- 2. 상품 데이터 (IT 기기 및 전문 서적)
INSERT INTO product(product_name, product_price, stock_quantity)
VALUES
('싱크패드 노트북', 3500000, 15),
('리얼포스 키보드', 380000, 40),
('델 모니터', 1200000, 25),
('JPA 프로그래밍 책', 32000, 150),
('로지텍 지슈라2 마우스', 139000, 200),
('로지텍 웹캠', 450000, 30),
('갤럭시 S25 휴대폰', 1800000, 50),
('아이폰 17', 1900000, 50);
-- 3. 주문 데이터 & 관련 데이터 (주문, 주문상품, 배송, 결제)
-- 시나리오 1: 세종대왕, 'JPA 프로그래밍 책' 도서 1권 주문 (배송 완료)
INSERT INTO orders(member_id, ordered_at, order_status, total_amount)
VALUES
(1, '2025-09-05 10:00:00', 'COMPLETED', 32000);
SET @last_order_id = LAST_INSERT_ID();
INSERT INTO order_item(order_id, product_id, product_name, order_price,
order_quantity)
VALUES
(@last_order_id, 4, 'JPA 프로그래밍 책', 32000, 1);
INSERT INTO delivery(order_id, delivery_status, tracking_no, ship_addr)
VALUES
(@last_order_id, 'COMPLETED', 'KR13970515', '서울시 종로구 경복궁');
INSERT INTO pay(order_id, pay_method, pay_amount, pay_status, paid_at)
VALUES
(@last_order_id, 'BANK_TRANSFER', 32000, 'PAID', '2025-09-05 10:05:00');
-- 시나리오 2: 이순신, 전투 지휘용 '델 모니터' 1대 주문 (배송중)
INSERT INTO orders(member_id, ordered_at, order_status, total_amount)
VALUES
(2, '2025-09-15 14:30:00', 'SHIPPING', 1200000);
SET @last_order_id = LAST_INSERT_ID();
INSERT INTO order_item(order_id, product_id, product_name, order_price,
order_quantity)
VALUES
(@last_order_id, 3, '델 모니터', 1200000, 1);
INSERT INTO delivery(order_id, delivery_status, tracking_no, ship_addr)
VALUES
(@last_order_id, 'SHIPPING', 'KR15450428', '전라남도 여수시 진남관');
INSERT INTO pay(order_id, pay_method, pay_amount, pay_status, paid_at)
VALUES
(@last_order_id, 'CARD', 1200000, 'PAID', '2025-09-15 14:32:00');
-- 시나리오 3: 네이트, '싱크패드 노트북'과 '리얼포스 키보드' 주문 (배송 준비)
INSERT INTO orders(member_id, ordered_at, order_status, total_amount)
VALUES
(4, '2025-09-20 09:00:00', 'ORDERED', 3880000);
SET @last_order_id = LAST_INSERT_ID();
INSERT INTO order_item(order_id, product_id, product_name, order_price,
order_quantity)
VALUES
(@last_order_id, 1, '싱크패드 노트북', 3500000, 1),
(@last_order_id, 2, '리얼포스 키보드', 380000, 1);
INSERT INTO delivery(order_id, delivery_status, ship_addr)
VALUES
(@last_order_id, 'READY', '서울시 송파구 잠실동');
INSERT INTO pay(order_id, pay_method, pay_amount, pay_status, paid_at)
VALUES
(@last_order_id, 'CARD', 3880000, 'PAID', '2025-09-20 09:01:00');
-- 시나리오 4: 스티브 잡스, '갤럭시 S25 휴대폰'을 주문했으나 취소
INSERT INTO orders(member_id, ordered_at, order_status, total_amount)
VALUES
(5, '2025-09-11 18:00:00', 'CANCELED', 1800000);
SET @last_order_id = LAST_INSERT_ID();
INSERT INTO order_item(order_id, product_id, product_name, order_price,
order_quantity)
VALUES
(@last_order_id, 7, '갤럭시 S25 휴대폰', 1800000, 1);
INSERT INTO delivery(order_id, delivery_status, tracking_no, ship_addr)
VALUES
(@last_order_id, 'SHIPPING', 'EA12341234', '미국 캘리포니아');
INSERT INTO pay(order_id, pay_method, pay_amount, pay_status)
VALUES
(@last_order_id, 'CARD', 1800000, 'CANCELED');
-- 시나리오 5: 션, 'JPA 프로그래밍 책'과 '로지텍 지슈라2 마우스' 주문 (배송 완료)
INSERT INTO orders(member_id, ordered_at, order_status, total_amount)
VALUES
(6, '2025-08-25 21:00:00', 'COMPLETED', 171000);
SET @last_order_id = LAST_INSERT_ID();
INSERT INTO order_item(order_id, product_id, product_name, order_price,
order_quantity)
VALUES
(@last_order_id, 4, 'JPA 프로그래밍 책', 32000, 1),
(@last_order_id, 5, '로지텍 지슈라2 마우스', 139000, 1);
INSERT INTO delivery(order_id, delivery_status, tracking_no, ship_addr)
VALUES
(@last_order_id, 'COMPLETED', 'KR19691228', '성남시 분당구 판교동');
INSERT INTO pay(order_id, pay_method, pay_amount, pay_status, paid_at)
VALUES
(@last_order_id, 'CARD', 171000, 'PAID', '2025-08-25 21:05:00');
-- 시나리오 6: 션, '싱크패드 노트북'과 '로지텍 웹캠' 추가 주문 (배송중)
INSERT INTO orders(member_id, ordered_at, order_status, total_amount)
VALUES
(6, '2025-09-22 13:10:00', 'SHIPPING', 3950000);
SET @last_order_id = LAST_INSERT_ID();
INSERT INTO order_item(order_id, product_id, product_name, order_price,
order_quantity)
VALUES
(@last_order_id, 1, '싱크패드 노트북', 3500000, 1),
(@last_order_id, 6, '로지텍 웹캠', 450000, 1);
INSERT INTO delivery(order_id, delivery_status, tracking_no, ship_addr)
VALUES
(@last_order_id, 'SHIPPING', 'KR20250922', '성남시 분당구 정자동');
INSERT INTO pay(order_id, pay_method, pay_amount, pay_status, paid_at)
VALUES
(@last_order_id, 'CARD', 3950000, 'PAID', '2025-09-22 13:11:00');
```
