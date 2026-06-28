# S06. SQL - 조회와 정렬

## 조회 실습 데이터 준비

---

## SELECT - 조회

### select * 주의사항
- 성능 저하: 불필요한 데이터까지 모두 읽어오느라 데이터베이스 시스템에 큰 부담을 줌. 조회 속도가 느림
- 가독성 저하: 내가 보고 싶은 데이터는 이름과 이메일뿐인데, 수십개의 열이 함께 표시되니 한 눈에 파악하기 어려움
- 네트워크 트래픽 낭비: 데이터베이스 서버에서 우리 컴퓨터로 데이터를 전송할 때, 필요 없는 데이터까지 함께 보내므로 네트워크 자원을 낭비하게 됨

### 열 이름에 별칭 붙이기: as 별칭
```sql
select name as 고객명, email as 이메일 from customers;
```
- 내가 원하는 열 이름으로 머릿글을 변환해서 볼 수 있음
- 장점
    - 보고서의 가독성 향상: 보고서로 뽑아볼 수 있음
    - 열 이름의 충돌 방지: 지금은 테이블 하나만 다루지만 나중에 여러 테이블을 연결해서 조회할 때, 서로 다른 테이블에 같은 이름의 열이 존재할 수 있음
    - 이때, 별칭을 사용하면 열을 명확하게 구분할 수 있음
- as `벡틱을 활용해서 무엇이든지 사용할 수 있음`

---

## WHERE - 기본 검색

### 행만 걸러내는 필터
```sql
select 열이름
from 테이블이름
where 조건;
```
- 조건(조건문)
    - 보통 비교 연산자를 사용하여 만듦 (프로그래밍 언어와 약간의 차이가 있음)
    - =: 같다
    - != 또는 <>: 같지 않다
    - 그 외는 동일함
    - 추가: is

> 참고: 날짜는 작은 따옴표로 감싸줌 / 숫자는 그대로 사용 

### 기본적인 조회
```sql
select * from products
where price >= 10000;
```

### 조금더 복잡한 조회 - AND, OR, NOT
- A AND B: A, B의 조건을 모두 만족하면 TRUE / FALSE
- A OR B: A, B 둘 중 하나만 만족하면 TRUE / FALSE
- NOT: 주어진 조건을 부정함 (IN, LIKE BETWEEN, NOT NULL 등과 함께 사용함)

---

## WHERE - 편리한 조건 검색

### BETWEEN: 특정 범위에 있는 값 찾기 (포함)
```sql
select * from products
where price >= 5000 and price <= 15000;

select * from products
where price betwenn 5000 and 15000;
```

### NOT BETWEEN: 특정 범위를 제외한 값 찾기 (포함 X)
```sql
select * from products
where price < 5000 or price > 15000;

select * from products
where price not between 5000 and 15000;
```

### IN: 목록에 포함된 값 찾기
```sql
select * from products
where name = '갤럭시' or name = '아이폰' or name = '에어팟';

select * from products
where name in ('갤럭시', '아이폰', '에어팟');
```

### NOT IN: 목록에 포함되지 않는 값 찾기
```sql
select * from products
where name != '갤럭시' and name != '아이폰' and name != '에어팟';

select * from products
where name not in ('갤럭시', '아이폰', '에어팟');
```

### LIKE: 문자열의 일부로 검색하기 (패턴 매칭)
- 와일드카드 문자
    - %: 뭐가 오든 된다라는 와일드 카드
        - -%: 앞글자는 고정
        - %-: 뒷글자는 고정
        - %-%: 중간 글자는 고정
    - _: 정확히 한 글자만 뭐가 와도 된다는 와일드 카드 
        - 이_신: 이로 시작하고 신으로 끝나는 "세글자"

```sql
select * from customers 
where email like '%sejong%';
```

### NOT LIKE: 특정 패턴을 제외하고 검색하기
```sql
select * from customers 
where address not like '서울특별시%';
```

---

## ORDER BY - 정렬

### 기본적인 사용법
```sql
select 열이름
from 테이블이름
where 조건
order by 정렬기준열 [정렬방식];
```
- 정렬방식은 생략 가능: ASC Default

### 정렬방식
- ASC: 오름차순 조건
- DESC: 내림차순 조건
```sql
select * from customers 
order by join_date DESC;
```

### 정렬 기준이 여러 개라면?
- 재고 수량 많은 순서
- 재고 수량이 같다면, 상품들끼리는 가격이 낮은 순서대로
- 1차 필터링, 2차 필터링, ...
```sql
select * from products
order by stock_quantity desc, price asc; 
```

---

## LIMIT 개수 제한

### 기본적인 사용법
```sql
select 열이름
from 테이블이름
where 조건
order by 정렬기준
limit 개수;
```
- 조회할 행의 개수를 제한하는 역할

### 주의사항
- limit은 order by랑 함께 사용해야지만 의미가 있음
- 데이터베이스는 결과의 순서를 보장하지 않음
```sql
select * from products
order by price desc
limit 2;
```

### 페이징 처리
- 건너뛸 개수와 가져올 개수를 알아야함
- limit offset, row_count
- offset: 건너뛸 개수
- row_count: 가져올 개수
```sql
select * from products
order by product_id asc
limit 2, 5;
```

### 페이징 적용 방법
- 한 페이지에 2개의 상품을 보여주는 상황
    - 1페이지: 0, 2
    - 2페이지: 2, 2
    - 3페이지: 4, 2

- 규칙
    - offset = (페이지 번호 - 1) * row_count;

---

## DISTINCT - 중복 제거
- 원하지 않는 중복된 값들이 나오는 경우가 있음
- 유일한 값만 깔끔하게 추리는 방법

### 기본적인 사용법
```sql
select distinct 열이름

select customer_id from orders; // 1 1 2 2 3

select distinct customer_id from orders; // 1 2 3

select customer_id, product_id from orders;

select distinct customer_id, product_id from orders;
```
- 열이름의 조합 자체가 중복되지 않게 가져옴

---

## NULL - 알 수 없는 값

### 잘못 사용하는 경우
```sql
select * from products
where description = null;
```

### IS, NOT IS
```sql
select * from products
where description is null;

select * from products
where description is not null;
```

### DB에서 NULL의 근본적인 의미
- 알 수 없음
- 값이 없음을 나타내는 특별한 상태
- 0이나 빈문자열과는 완전히 다른 의미
- 따라서 =으로 비교할 수 없음 (값 자체가 없기 때문)
- "알 수 없음(UNKNOWN)이라는 값을 반환함

### NULL 정렬
- MYSQL은 null을 가장 작은 값으로 취급함
- 이건 데이터베이스 정책에 따라서 다름

### NULL의 위치를 강제로 바꾸고 싶을 때
- 상품 설명을 내림차순으로하되, 설명이 없는 상품은 빨리 확인할 수 있게 맨 앞으로 보내주세요
```sql
select product_id, name, description, description is null from products
order by description is null desc, description desc;
```
- description is null이라는 항목 자체를 기준으로 정렬을 시도
    - description is null은 true면 1, false는 0
- 이후 description으로 desc
