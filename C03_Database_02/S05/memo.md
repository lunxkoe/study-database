# S05. UNION

## UNION
- 조인, 서브쿼리라는 강력한 도구를 배움
- 이 기술들의 공통점은 기존 테이블의 정보를 조합하거나 필터링해서, 우리가 원하는 형태의 하나의 결과 집합을 만들어내는 것
- 조인이 옆으로 더 많은 정보를 가진 컬럼을 만드는 기술이였다면, 
- UNION은 여러 개의 결과 집합을 아래로 붙여서 더 많은 행을 가진 하나의 집합으로 만드는 기술

### 문제 상황
- 현재 활동 중인 고객을 users 테이블에, 과거 탈퇴한 곡갤을 retired_users라는 별도의 테이블에 보관하고 있음
- 연말을 맞아 모든 고객(활동 + 탈퇴)에게 감사 이메일을 보내기 위해, 두 테이블에 흩어져있는 이름과 이메일을 합쳐서 하나의 전체 목록을 만들어야함
- 이것은 조인으로 해결할 수 없음
    - 연결된 관계가 아니라, 구조는 비슷해도 분리된 별개의 집합이기 때문

### 개념과 사용법
```sql
select name, email from users
union
select name, email from retired_users;
```

### 핵심 규칙
- union으로 연결되는 모든 select 문은 컬럼의 개수가 동일해야함
- 같은 위치에 있는 컬럼들은 서로 호환 가능한 데이터 타입이어야함!!
- 최종 결과의 컬럼 이름은 첫 번째 select문의 컬럼 이름을 따름

### 동작
- 션은 users / retired_users에도 있음
- 근데 현재 최종 결과 목록에서는 하나만 있음

- 유니온은 중복된 행은 자동으로 제거하고 고유한 값만 남김!!

- **의도적으로 중북을 허용하거나, 데이터에 중복이 없다는 것을 이미 알고 있다면 union all을 사용하면 됨

--- 

## UNION ALL

### 문제 상황
- 마케팅 팀에서 두 종류의 고객에게 이벤트 안내 메일을 보내려고 함
- 첫 번째 그룹은 "전자기기" 카테고리의 상품을 구매한 이력이 있는 고객
- 두 번째 그룹은 "서울"에 거주하는 고객
- 두 그룹의 명단을 합쳐서 전체 발송 목록을 만들고 싶음

- 질문
    - 서울에 살면서 전자기기를 구매한 고객은 두 그룹에 모두 속하게 됨
    - 이 고객을 최종 목록에 한 번만 포함해야할까? 중복을 허용해도될까?
    - **비즈니스 요구사항에 따라 다름**

### UNION / UNION ALL의 차이
- 중복 처리 여부
    - UNION: 두 결과 집합을 합친 후, 중복된 행을 제거
    - UNION ALL: 중복 제거 과정 없이 두 결과 집합을 그대로 모두 합침

- UNION 사용 시(중복 제거)
```sql
select u.name, u.email
from users u
join orders o on u.user_id = o.user_id
join products p on o.product_id = p.product_id
where p.category = '전자기기'

union

select u.name, u.email
from users u
where u.address like '서울%';
```

- UNION ALL(중복 미제거)
```sql
select u.name, u.email
from users u
join orders o on u.user_id = o.user_id
join products p on o.product_id = p.product_id
where p.category = '전자기기'

union all

select u.name, u.email
from users u
where u.address like '서울%';
```

### 선택 가이드
- 중복을 제거해야만 하는 명확한 요구사항 -> UNION
- 그 외 모든 경우는 UNION ALL(성능이 더 좋음)

---

## UNION 정렬

### Order By의 위치가 중요
- 전체를 하고 마지막에 딱 한 번만 사용할 수 있음
```sql
select name, email from users
union
select name, email from retired_users
order by name;
```

### UNION에 나오지 않는 필드를 사용한다면?
- **order by절에서는 첫 번째 select문의 컬럼 이름이나 해당 컬럼의 별칭만 사용할 수 있음**

---

## 문제와 풀이

### 전체 고객 목록 조회하기
```sql
select name, email from users 
union
select name, email from retired_users;
```

### 특별 이벤트 대상자 목록 만들기 (중복 포함)
```sql
select 
	distinct u.name, u.email
from orders o
join users u
	on o.user_id = u.user_id
join products p
	on o.product_id = p.product_id 
where p.category = '전자기기'

union all
	
select 
	distinct u.name, email -- (name, email)에 distinct가 적용되는 것
from orders o
join users u
	on o.user_id = u.user_id
where o.quantity >= 2;
```

### 회사 주요 이벤트 타임라인 만들기
```sql
select 
	created_at as `이벤트 날짜`,
	'고객 가입' as `이벤트 종류`,
	name as `상세 내용`
from 
	users

union all

select 
	o.order_date,
	'상품 주문',
	p.name
from 
	orders o
join products p
	on o.product_id = p.product_id
order by `이벤트 날짜` desc;
```

### 회사 전체 인명록 만들기
```sql
select 
	name as `이름`,
	'고객' as `역할`,
	email as `이메일`
from
	users

union all

select 
	name,
	'직원',
	concat(name, '@my-shop.com')
from employees

order by `이름`;
```
