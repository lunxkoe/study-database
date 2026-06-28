# S07. SQL - 데이터 가공

## 산술 연산

### SELECT 절 안에서 사칙연산 활용하기
```sql
select 
    name, 
    price, 
    stock_quantity, 
    price * stock_quantity as total_stock_value 
from products;
```
- 계산한 열은 별칭을 붙여주는 것이 좋음!!

---

## 문자열 함수

### 필요한 이유
- 보고서에서는 이름과 이메일을 합쳐서 하나의 문자열로 보기 좋게 표시하고 싶음
> 이순신 (yisunsin@example.com)

### 해결책: CONCAT() 함수로 문자열 합치기
```sql
select concat(name, '(' , email, ')') as `이름(이메일)` from customers;
```

```sql
select concat_ws(' - ', name, email, address) as customer_details from customers;
-- 중간 중간에 맨 앞에 것을 넣어줌
```

### 대소문자로 변환: UPPER(), LOWER()
```sql
select email, UPPER(email) as upper_email from customers;
```

### 문자열 길이 계산
```sql
select name, char_length(name) as name_length from customers;

select name, length(name) as byte_length from customers;
-- UTF-8 기준 한글은 3바이트
```

---

## NULL 값을 다루는 함수들

### IFNULL(표현식1, 표현식2)
- 표현식1: NULL인지 아닌지 검사할 컬럼이나 값
- 표현식2: 표현식1이 NULL일 경우 대신 반환할 값 
- 결과: 표현식1이 NULL이 아니면 표현식1의 값을 그대로 반환하고, NULL이면 표현식2의 값을 반환함
```sql
select
    name,
    ifnull(description, '상품 설명 없음') as description
from
    products;
```

### colesce(표현식1, 표현식2, ...)
- 인자 중, 처음으로 null이 아닌 값을 반환
- 모든 인자가 null이면 null을 반환
```sql
select
    name,
    coalesce(description, '상품 설명 없음') as description
from
    products;
```
- ifnull보다 유용한 상황
    - 짧은 설명과 긴 설명이 있을 경우 짧은 설명이 있으면 그것을 쓰고, 없으면 긴 설명을 쓰고, 그것도 없으면 설명없음을 표시하고 싶은 경우

---

## 다양한 함수 소개

### SQL 표준 함수 소개
- 표준 함수는 관계형 데이터베이스에서 거의 동일하게 동작하는 함수들을 말함
- 호환성에도 좋음
- 있다는 것을 알고 찾아보면 됨 (외우지 말자!!)