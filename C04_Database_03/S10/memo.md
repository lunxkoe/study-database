# S10. 정규화

## 정규화 - 시작

### 함수 종속성
- 테이블에서 컬럼의 값들이 다른 컬럼의 값을 유일하게 결정하는 관계를 의미함
- X -> Y
- **X가 Y를 함수적으로 결정한다**라고 읽음
- X: 결정자 / Y: 종속자

- 예
    - member_id -> member_name: member_id가 1이면 member_name은 항상 션
    - member_id -> member_address: member_id가 1이면 member_address는 항상 서울시 송파구

### 핵심
- 함수 종속성 관계를 분석해서, 잘못된 종속 관계를 찾아내고 테이블을 분리하여 올바른 종속 관계를 만들어가는 과정

### 용어
- 테이블 -> 릴레이션
- 행 -> 튜플
- 컬럼 -> 애트리뷰트

---

## 제 1 정규형

### 필요성
- 데이터 검색의 어려움
- 데이터 수정의 복잡성
- 데이터 타입 사용 불가

### 정의와 적용
- **테이블의 모든 컬럼이 원자적인 값만을 가져야함**
- 한 컬럼 안에 상품 ID, 상품명, 수량, 가격 등 이 포함되엉 있다면 여러 정보로 쪼갤 수 있음
    - product_info: 10:노트북:2:1500000, 20:휴대폰:1:300000
    - product_id: 10
    - product_name: 노트북
    - order_quantity: 2
    - order_price: 15000000
    - ordered_at

    - 추가로 (order_id, product_id)를 PK로 만듦

### 문제점
- 여전히 데이터 중복 문제는 해결이 되지 않음

---

## 제 2 정규형

### 필요성
- 현재 기본 키는 (order_id, product_id) 복합키
- 즉, 주문 ID와 상품 ID가 하나의 행을 고유하게 식별할 수있음

- 함수 종속 관계
    - 기본키 모두에 종속되는 컬럼
        - (order_id, product_id) -> order_price
        - (order_id, product_id) -> order_quantity

    - 기본 키의 일부에만 종속되는 컬럼
        - order_id -> member_id
        - order_id -> member_name
        - order_id -> ordered_at

        - product_id -> product_name, product_price

- 이처럼 기본 키의 일부에만 종속되는 컬럼이 존재하는 것을 **부분 함수 종속**이라고 함
    - 데이터 중복 문제
    - 갱신 이상
    - 삽입/삭제 이상


### 정의
- 제 1 정규형을 만족해야함
- 테이블의 모든 컬럼이 기본 키에 대해 완전 함수 종속이어야함
    - 즉, 부분 함수 종속이 없어야함

### 적용
- order_id => orders 테이블
- prouct_id => products 테이블
- order_item 연결 테이블

### 문제
- 했는데고 션이 두 번 나옴(member_name)

---

## 제 3 정규화

### 필요성
- 현재 션이라는 회원(member_name)이 중복되서 나오고 있음
- 함수 종속 관계 분석
    - member_id와 ordered_at은 기본 키인 order_id에 잘 종속되어있음
    - member_name은 기본 키가 아닌 member_id에 종속됨
    - 즉, order_id -> member_id -> member_name과 같이 종속됨
        - **이행적 함수 종속**

### 발생하는 문제
- 갱신 이상: 션 회원이 김개발로 개명하면 member_id가 1인 모든 회원의 주문을 바꿔야함
- 삽입 이상: 아직 주문을 한 번도 하지 않은 신규 회원은 orders 테이블에 등록할 수 없음(order_id가 없음)
- 삭제 이상: 1002 주문을 삭제했는데, 네이트 회원의 유일한 주문이였다면 네이트 회원 정보 자체가 사라짐

### 정의
- 제 2 정규형을 만족해야함
- 이행적 함수 종속이 존재하지 않아야 함

### 적용
- member_id -> member_name
- members 테이블로 변환

### 결론
- 이제 데이터 이상 현상이 모두 해결됨

---

## BCNF 정규형

### BCNF
- Boyce-Codd Normal Form
- **테이블의 모든 결정자가 후보 키여야 한다**
- 결정자는 X->Y에서 X
- 후보 키는 테이블의 행을 유일하게 식별할 수 있는 컬럼의 최소 집합
- 기본 키는 여러 후보 키 중 하나를 선택한 것

- 기본 키가 아니더라도 어떤 컬럼이 다른 컬럼을 결정한다면 반드시 후보 키여야한다는 것
- 대부분의 경우 제 3 정규형을 만족하며 BCNF도 만족함

### 필요성: 3NF의 한계
- (student_id, lecture_name, professor_name)

- 함수 종속성 분석
    - (student_id, lecture_name) -> professor_name을 알 수 있음
    - professor_name -> lecture_name: 교수의 이름을 알면 그가 담당하는 특강 이름을 알 수 있음
        - 한 교수는 오직 하나의 특강만 담당한다의 규칙이 있는 경우

- 후보 키 분석
    - (student_id, lecture_name)
    - (sutdent_id, professor_name)
        - professor_name -> lecture_name

- 정규형 만족 여부
    - 제 1 정규형 만족
    - 제 2 정규형 만족
        - 기본키는 (student_id, lecture_name)
        - 기본키가 아닌 컬럼 professor_name, 기본 키 전체에 종속됨
    - 제 3 정규형
        - 기본키가 아닌 컬럼이 professor_name 밖에 없어서 이행적 함수 종속이 없음

    - BCNF:
        - (student_id, lecture_name)은 professor_name을 결정함
        - professor_name은 lecture_name을 결정함
            - 근데 이거는 후보키가 아님
            - 따라서 BCNF를 위반함

### BCNF를 위반했을 때 문제점
- 갱신 이상: 서교수가 담당하는 특강이 자바에서 파이썬으로 변경된다면 서교수가 등장하는 모든 행의 lecture_name을 파이썬으로 바꿔야함
- 삽입 이상: 최교수가 인공지능 특강을 새로 개설했지만 신청한 학생이 없다면 테이블에 추가할 수 없음
- 삭제 이상: 102번 학생이 데이터베이스 수강을 취소하여 해당 행이 삭제되면 교수가 데이터베이스를 담당한다는 소중한 정보가 없어져버림

### 적용: 테이블 분리
- professor_name -> lecture_name
    - professors 테이블로 분리

- 
    - student_id
    - professor_name

- 
    - professor_name
    - lecture_name

---

## 실무와 정규화
- 정규화를 하는 이유
    - 데이터 중복 최소화
    - 데이터 일관성 및 무결성 확보
    - 데이터 이상 현상 해결
    - 유연한 데이터 구조

- 순서
    - 제 1 정규화
    - 제 2 정규화
    - 제 3 정규화
    - BCNF

### 정규화가 필요한 이유
- 하나의 칸에는 하나의 정보만
- 하나의 주제는 하나의 묶음으로
- 반복되는 정보는 한 곳에서 관리

### 제4, 5 정규화가 있음
- 실무에서는 3NF / BCNF까지 이해하면 됨
