# S09. 인덱스 2

## 옵티마이저와 인덱스 선택

### 문제 상황
- 컬럼에 인덱스를 사용하면, 해당 컬럼을 조건으로 사용하는 모든 where절의 성능이 향상될 것이라고 생각할 수 있음
- 하지만 데이터베이스 옵티마이저가 쿼리 실행 전에 평가를 하고, 가장 저비용이고 효율적인 방법을 선택함
- 인덱스 사용 효율 < 풀 테이블 스캔 효율일 수도 있음

### 어떤 경우일까?
- 하드웨어도 고려해야함
    - 하드웨어의 경우 근처에 있는 데이터를 쭉 읽어들이는게 더 빠를 때도 있음

### 인덱스 손익분기점
- 옵티마이저가 인덱스 사용 여부를 결정하는 핵심 기준은 바로 손익분기점
- 여기서는
    - 인덱스를 통해 데이터를 읽는 비용이 테이블 전체를 직접 읽는 비용보다 높아지는 지점을 의미
    - 인덱스를 사용하는 비용
        - 인덱스 탐색 비용 + 인덱스에서 찾은 주소로 테이블에 접근하는 비용(랜덤 I/O)
    - 풀 테이블 스캔 비용
        - 테이블 전체를 순차적으로 읽는 비용(순차 I/O)

- 일반적으로 전체 데이터의 약 20% - 25% 이상을 조회해햐하는 쿼리는 인덱스 보다, 테이블 전체를 순차적으로 스캔하는 것이 더 효율적이라고 알려져 있음

### 예시 1: 인덱스를 사용하는 효율적인 범위 검색
```sql
explain select * from items where price between 50000 and 100000;
```
- 25건 중 5개

### 예시 2: 인덱스를 포기하는 비효율적인 범위 검색
```sql
explain select * from items where price 1000 and 200000;
```
- possible key는 있는데,
- type은 all: 풀 테이블 스캔

> 여러 인덱스가 있다면?
- 그 중에서 가장 효율적으로 작동하는 인덱스를 선택
- 풀테이블스캔도 고려함

### 데이터가 많이 부족하다면?
- 풀 테이블 스캔을 선택할 가능성이 매우 높음
- 강제로 사용하고 싶다면
    ```sql
    explain select * from items 
    force index(idx_items_price)
    where price between 50000 and 100000;
    ```

---

## 커버링 인덱스

### 옵티마이저가 인덱스를 사용하기로 결정해도 추가적인 작업이 필요
- 인덱스 스캔: idx_items_price 인덱스에서 where 조건에 맞는 데이터의 위치를 찾음
- 테이블 데이터 접근: 인덱스에서 찾은 위치 정보를 사용해, 원본 테이블에 접근해서 select 절에서 요구하는 다른 컬럼의 데이터를 가져옴

- 핵심: **원본 테이블에 다시 접근하는 과정은 랜덤 I/O를 유발함**
- 그렇다면
    - 쿼리에 필요한 모든 데이터를 인덱스에서만 읽어서 끝낼 수는 없을까?
    - **커버링 인덱스**

### 커버링 인덱스란
- 쿼리에 필요한 모든 컬럼을 포함하고 있는 인덱스를 의미함
- 그렇다면 원본 테이블에 접근하지 않아도 됨
    - 이는 랜던 I/O 작업을 제거하고 성능이 비약적으로 증가함

### 커버링 인덱스 적용전
```sql
select item_id, price, item_name from items where price between 50000 and 100000;
explain select item_id, price, item_name from items where price between 50000 and 100000;
```
- 위 쿼리를 실행하면 인덱스가 걸려있음
- 원본 테이블에 접근해야하기 때문에 랜덤 I/O가 발생함
- 참고로 
    - MySQL의 인덱스는 테이블의 기본키(item_id)를 기본으로 함

### 커버링 인덱스 적용 - 인덱스 컬럼만 조회하는 경우
```sql
select item_id, price from items where price between 50000 and 100000;
explain select item_id, price from items where price between 50000 and 100000;
```
- Extra: Using where; **Using index**
    - using index
    - 쿼리에 필요한 모든 데이터를 인덱스 테이블만 사용했다는 의미!!!
    - 또한 using index와 같이 사용되는 using where는 
        - 테이블에 접근한 후 필터링하는 것이 아니라
        - 인덱스 스캔 단계에서 효율적으로 필터링이 이루어졌음을 나타냄

### 커버링 인덱스 적용 - item_name 추가
- item_name을 포함하는 쿼리의 성능을 최적화
- 이 쿼리에 필요한 컬럼은 where절의 price와 select절의 item_name
- 이 두 컬럼을 모두 포함하는 인덱스를 생성하면 됨
> 컬럼이 여러 개인 **복합 인덱스**에서 컬럼의 순서는 매우 중요함
- where 절에서 동등비교나 범위 검색에 사용되는 컬럼을 가장 앞에 두어야 인덱스를 효율적으로 사용할 수 있음
    - price, item_name 순으로 해야함
```sql
drop index idx_items_price on items;
create index idx_items_price_name on items (price, item_name);

explain select item_id, price, item_name
from items
where price between 50000 and 100000;
```
- Extra: using where; using index

### 어떻게 생긴 걸까?
- 구조
    - price | item_name | 원본 위치(item_id)

- price로 먼저 접근하고, item_name, item_id를 가져오면 됨

### 커버링 인덱스의 장단점
- 장점
    - 압도적인 select 성능 향상
    - 특히 count 쿼리 최적화: select count(*)와 같은 쿼리에서 테이블 전체가 아닌, 크기가 훨씬 작은 인덱스만 스캔하여 결과를 반환

- 단점
    - **저장 공간 증가**: 컬럼이 많아질수록 인덱스의 크기도 커짐
    - **쓰기 성능 저하**: insert, update, delete 작업 시, 테이블 데이터뿐만 아니라 인덱스도 함께 수정해야함으로 쓰기 작업에 대한 부하가 커짐

### 언제 사용할까?
- 만능이 아님
    - 조회가 빈번하고, 쓰기 작업이 상대적으로 적은 테이블에 주로 사용
    - 또한 조회하는 컬럼의 개수가 적을 때 유리
    - 성능 저하가 발생하는 특정 쿼리를 튜닝하기 위한 비장의 무기로 사용하는 경우가 많음!!

---

## 복합 인덱스 1

### 상황
- 실제 상황에서는 여러 조건으로 검색하는 경우가 매우 많음
- 이때 사용하는 것이 복합 인덱스(다중 컬럼 인덱스)라고 함

### 복합 인덱스
- 두 개 이상의 컬럼을 묶어서 하나의 인덱스로 만드는 것
- 가장 중요한 점
    - **컬럼의 순서!!**

### 왜 컬럼의 순서가 중요할까?
- 복합 인덱스 동작 원리는 전화번호부나 국어사전과 같음
    - 전화번호부: 성으로 먼저 정렬된 후, 같은 성 안에서 이름으로 다시 정렬됨
    - 국어사전: 첫 번째 글자로 먼저 정렬된 후, 같은 첫 글자로 시작하는 단어들끼리 두 번째 글자로 다시 정렬됨

- item 테이블 예
    - (category, price)
    - category를 기준으로 먼저 정렬함
    - 같은 category 내에서 price를 기준으로 다시 정렬함
    - category | price | item_id

> 2차 정렬로만은 정렬이 유지되지 않음

- 이처럼 복합 인덱스는 첫번째 컬럼을 기준으로 정렬된 상태에서만 제 역할을 할 수 있음
- **매우 유의**
    - 인덱스를 (a, b, c)로 했다면
    - where에 사용할 때 (a) (a, b) (a, b, c)로 해야함
        - 순서를 의미하는 것이 아님!!
        - 그냥 a가 포함되어야함고
        - (a, c)이런건 안된다는 거임

> 복합 인덱스 대원칙
- 인덱스는 순서대로 사용해라(왼쪽 접두어 규칙)
- 등호 조건은 앞으로, 범위 조건(<, >)은 뒤로
- 정렬도 인덱스 순서를 따르라!!!
- **A가 빠진 조건으로는 인덱스를 제대로 활용할 수 없음!!

### 복합 인덱스 준비 과정
- 기존에 있던 인덱스를 전부 다 제거해야함
- 기존 인덱스를 사용할 수도 있음
```sql
create index idx_items_category_price on items(category, price);
```

### 복합 인덱스 성공 예제1: category 사용
- 카테고리가 전자기기인 모든 상품을 찾기
```sql
select * from items where category = '전자기기';
explain select * from items where category = '전자기기';
```
- type: ref: category가 전자기기인 조건을 만족하는 = 사용
- key: idx_items_category_price
- rows: 10

### 복합 인덱스 성공 예제 2: category, price 함께 사용
```sql
select * from items where category = '전자기기' and price = 120000;
explain select * from items where category = '전자기기' and price = 120000;
```
- type: ref
- rows: 1

> 오해: where price = 1200000 and category = '전자기기' 순서를 바꿔도 됨

### 복합 인덱스와 성공 예제 3: 복합 인덱스와 정렬
- 복합 인덱스의 진정한 힘은 정렬 작업을 피할 때 드러남
- where절의 필터링과 order by의 정렬 방향이 인덱스 순서와 일치하면 불필요한 filesort 작업을 생략함
```sql
explain select * from items where category = '전자기기' and price > 10000;
explain select * from items where category = '전자기기' and price > 10000 order by price;
```
- filesort를 하지 않는 것을 알고 있음
- 전자기기 안에서 가격으로 정렬이 이미 되어있음
- 오해
    - category, price의 필터링 순서는 중요하지 않음
    - 인덱스를 설정할 때, =을 사용하는걸 먼저 하라는 의미임

```sql
explain select * from items where category = '전자기기' and price > 10000 order by item_name;
```
- using filesort를 수행함(item_name은 없기 때문)

---

## 복합 인덱스 2

### 문제 상황
- 첫 번재 컬럼(category)를 건너뛰고, 두 번째 컬럼(price)만으로 데이터를 검색하는 경우
```sql
select * from items where price = 80000;
explain select * from items where price = 80000;
```
- type: all
- key: null

### 복합 인덱스 실패 예제 2: 범위 조건을 먼저 사용
- 선행 컬럼에 범위 조건을 사용하면, 뒤에 오는 컬럼은 인덱스를 제대로 활용할 수 없음
```sql
select * from items where category >= '패션' and price = 20000;
explain select * from items where category >= '패션' and price = 20000;
```
- type: range
- key: idx_items_category_price
- 어 사용했네 하고 넘어가면 안됨
- 인덱스를 사용한 건 맞음
- 근데 filtered가 10
    - 10퍼센트를 남기고 날린다고?

- 왜 그럴까?
    - 패션 ~ 헬스/뷰티를 범위롤 스캔함
    - 근데 범위를 스캔 했는데, 문제는 가격을 찾아야함으로 결국 날려야함
    - 즉, 카테고리를 범위로는 성공함
    - 근데 결국 가격으로 속아내야함
        - 즉, 인덱스 사용이 아닌 결국 필터링을 하는 것임

### 결론
- 앞선 컬럼에 범위 조건이 걸리는 순간 정렬이 깨져버림
- 결국 필터링을 수행할 수 밖에 없음

---

## 복합 인덱스 3

### 범위 검색은 마지막에 한 번만 사용
```sql
select * from items where category >= '패션' and price = 20000;
explain select * from items where category >= '패션' and price = 20000;
```
- 이걸 많이 사용한다면?
    - (price, category)로 인덱스를 변경해야함

### price_category로 해보기
```sql
create index idx_items_price_category_temp on items (price, category);
select * from items where category >= '패션' and price = 20000;
explain select * from items where category >= '패션' and price = 20000;
```
- Extra: using index condition
- rows: 1
- filtered: 100

### 기존 인덱스를 잘 활용하자
- 인덱스를 계속 만드는 것은 만능이 아님
- 인덱스를 추가하면 그만큼 관리 비용이 들어감
- 기본적으로 기존 인덱스를 최대한 잘 활용하고
- 그래도 안되면 인덱스 추가를 고려해야함!!

### 사실 idx_item_category_price로도 최적화할 수 있음
- **IN 절 활용하기**

### IN 절 활용하기
- MySQL 옵티마이저는 IN()을 하나의 큰 범위로 취급하지 않고, 
- 여러 개의 동등 비교 조건의 묶음으로 인식함
- category >= '패션' == category in ('패션', '헬스/뷰티')
```sql
select * from items where category in ('패션', '헬스/뷰티') and price = 20000;
explain select * from items where category in ('패션', '헬스/뷰티') and price = 20000;
```
- filter: 100

### 동작 원리
- where category = '패션' or category = '헬스/뷰티'로 내부적으로 만들어짐
- 비유하자면
```sql
select * from items where category = '패션' and price = 20000
union all
select * from items where category = '헬스/뷰티' and price = 20000
```

> 팁
- 실무에서는 범위가 한정적인 컬럼에 이 IN절 트릭을 자주 활용함

--- 

## 복합 인덱스 정리

### 대원칙
- 인덱스는 순서대로 사용해라
- 등호 조건은 앞으로, 범위 조건은 뒤로
- 정렬도 인덱스 순서를 따르자

### 정렬도 인덱스 순서를 따르라!
- where category = '전자기기' order by stock_quantity
    - 이때 category_stock_quantity로 만들자

---

## 인덱스 설계 가이드라인

### 중요한 것
- 어디에 인덱스를 만들어야하는지 아는 것이 좋음
- 잘못된 인덱스는 오히려 시스템 성능을 떨어뜨릴 수 있음
- 인덱스는 결코 공짜가 아님

### 핵심 원칙: 카디널리티
- 해당 컬럼에 저장된 값들의 고유성 정도를 나타내는 지표
- 카디널리티가 높음
    - 해당 컬럼에 중복되는 값이 거의 없음
    - item_id, item_name
- 카디널리티가 낮음
    - category(5종류), is_active(2종류)

- 인덱스는 결국 찾아보기
    - 특정 키워드를 찾았을 때 검색의 범위가 확 줄어야함!!
    - 예
        - is_active를 true로 검색했는데 80%가 is_active일 경우
        - 결국 80%를 스캔해야함
    - 결론: **카디널리티가 높은, 즉, 식별력이 좋은 컬럼에 생성할 때 가장 효율적임**


### 인덱스 생성 가이드라인
1. where 절에 자주 사용되는 컬럼
2. join의 연결고리가 되는 컬럼(외래 키)

> 조인의 논리적인 순서와 실제 순서의 차이
- SQL의 논리적인 순서로 보면 join을 하고 where을 하지만
- 최적화를 위해 먼저 where로 필터링을 하고 join을 함
- 결과는 동일함을 보장함

> Join에 사용되는 외래 키 컬럼에는 반드시 인덱스를 생성
- MySQL은 자동으로 생성해줌

3. Order By 절에서 자주 사용되는 컬럼
- 정렬은 데이터가 많을 경우 매우 비용 큰 작업
- 최신 등록 상품 목록 10개를 보여주는 order by registered_date desc limit 10과 같은 쿼리는 registerd_date 컬럼에 인덱스가 있을 때 엄청난 성능 향상을 기대할 수 있음

---

## 인덱스의 단점과 주의사항

### 결론
- 데이터베이스 성능을 망치는 최악의 선택!!

### 인덱스는 공짜가 아님. 인덱스의 단점
1. 저장 공간
- B-Tree 구조를 가진 물리적인 파일로 디스크에 생성됨
- 추가 저장 공간이 필요함

2. 쓰기 성능 (INSERT / UPDATE / DELETE)
- **데이터 변경이 일어날 때마다 모든 인덱스가 수정되어야함**

### 실무 가이드: 균형의 미학
1. 워크로드를 분석하라: 읽기 vs 쓰기
    - 읽기 중심 서비스: 데이터 분석 시스템, 블로그, 뉴스 사이트처럼 데이터 변경보다는 조회가 훨씬 빈번한 서비스라면, 다양한 조회 성능을 높이기 위해 비료적 자유롭게 인덱스를 생성해도 좋음

2. 혹시나 해서 인덱스를 만들지마라
    - 사용하지 않는 인덱스는 저장 공간만 차지하고 쓰기 성능만 저하시키는 암적인 존재
    - 명확한 목적 없이 만들지말고
    - 느린 쿼리가 발견되면 그 쿼리를 개선하기 위한 목적으로 생성해야함

3. 사용하지 않는 인덱스는 주기적으로 정리해라
- 특정 인덱스가 얼마나 사용되었는지 모니터링하는 기능을 제공함
- 사용하지 않는 인덱스가 있다면 과감하게 삭제하자

---

## 문제와 풀이

### 인덱스를 만들어서 다음 쿼리 성능을 개선
- 문제 쿼리
```sql
SELECT * FROM items
WHERE category = '전자기기' AND is_active = TRUE;

SELECT * FROM items
WHERE category = '전자기기' AND is_active = TRUE
ORDER BY stock_quantity DESC;

SELECT * FROM items
WHERE stock_quantity < 90 AND category = '전자기기' AND is_active = TRUE;

SELECT * FROM items
WHERE stock_quantity < 90 AND category = '전자기기' AND is_active = TRUE
ORDER BY stock_quantity DESC;
```

### 정답
```sql
create index idx_items_category_is_active_stock_quantity_reverse on items(category, is_active, stock_quantity desc);
```
- 하나의 쿼리만 보고 만드는 것이 아니라 전체 쿼리를 보고 분석해야함
- 그외 추가적으로 얼마나 자주 사용하는지 서비스의 목적이 뭔지 등등 추가 고려해야함

---

## 정리

### 옵티마이저와 인덱스 선택
- 인덱스 효율이 좋지 않으면 옵티마이저가 풀 테이블 스캔을 선택할 수 있음
- 인덱스 사용을 강제할 수도 있음

### 커버링 인덱스
- 원본 테이블에 접근하지 않고, 인덱스 자체에서만 데이터를 가져옴
- Extra: using index

### 복합 인덱스 대원칙
- 인덱스는 순서대로 사용해라
- 등호는 앞으로, 범위는 뒤로
- 정렬도 인덱스 순서를 따르자

### 주의점
- 선행 컬럼 조건에 범위를 사용하면 효율이 떨어짐
- 범위 대신 in절을 사용하도록 하자 (새로 만드는 것도 고려할 수 있음)

### 가이드라인
- 카디널리티가 커야함(중복도가 낮아야함)
- where절에서 자주 사용되는 컬럼
- join의 연결고리가 되는 컬럼
- order by절에서 자주 사용되는 컬럼(정렬 작업 회피)


## 단점과 주의사항
- 저장 공간 차지
- 쓰기 성능 저하
- **읽기 중심 서비스와 쓰기 중심 서비스를 구분해야함**
