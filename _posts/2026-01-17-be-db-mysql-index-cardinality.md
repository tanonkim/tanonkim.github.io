---
title: "MySQL Index with Cardinality"
date: 2026-01-17 16:25:00 +0900
categories: [database, mysql]
tags: [mysql, index]
---

### Index

인덱스란 데이터의 저장(INSERT, UPDATE, DELETE) 의 성능을 희생하고 그 대신에 데이터의 읽기 속도를 높이는 테이블의 동작속도(조회)를 높여주는 자료구조이다.

인덱스가 없더라도 데이터베이스를 '작동' 하는데 있어서는 문제는 없다. 하지만 데이터베이스의 크기가 크면 클 수록, 인덱스가 반드시 필요해진다. 인덱스는 데이터베이스의 성능(속도)를 크게 좌우하는 요소이기 때문이다.<br>어떤 항목을 select 하는데 있어 몇분이 걸리던것이 인덱스를 이용해 조회하면 0.초 단위로 끝낼수 있을 정도이다.

인덱스 자체 역시 하나의 데이터 덩어리이기 때문에 전체 크기의 10%의 추가적인 공간을 할당해야 하고, 잘못 사용하는 경우 성능이 오히려 크게 떨어질 수 있는 단점이 있다. (변경 작업이 자주 일어나는 경우, 인덱스가 적절하지 않는 경우) → 개발자는 효율적인 인덱스 설계로 단점을 최대한 보완하는 구조 방법을 끊임없이 고민해야 한다.

- select 검색 속도를 크게 향상 시킨다.
- 인덱스 생성 시 DB 크기의 약 10% 정도되는 추가 공간이 필요하다.
- 인덱스 생성 시 시간이 걸린다. (몇분 씩 걸리기도 한다)
- insert, update, delete같은 데이터 변경 쿼리가 잦은 경우 paging이 빈번해져 성능이 악화될 수 있다.
- 데이터 조회에는 플러스지만, 데이터 변경이 자주 일어나면 오히려 성능 감소된다.

### 인덱스 생성

```sql
CREATE TABLE books (
  -- 같이 지정
  id varchar(5) primary key, -- 기본키 지정 (클러스터 인덱스)
  name varchar(20) unique, -- 인덱스 생성 (보조 인덱스) (중복 비허용)
  writer varchar(20) NOT NULL,
  
  INDEX idx_test (writer asc) -- 인덱스 생성 (보조 인덱스)
);
```

```sql
CREATE TABLE books (
  id varchar(5) primary key,
  name varchar(20) NOT NULL,
  writer varchar(20) NOT NULL,
  
  -- 뒤에 따로 할당
  [CONSTRAINT 제약조건이름] PRIMARY KEY (id), -- 기본키 지정 (클러스터 인덱스)
  [CONSTRAINT 제약조건이름] unique (id) -- 인덱스 생성 (보조 인덱스) (중복 비허용)
  INDEX idx_test (writer asc) -- 인덱스 생성 (보조 인덱스)
);
-- INDEX <인덱스명> ( 칼럼명1, 칼럼명2 )
```

### 인덱스 설계

- **WHERE 절 기준으로 인덱스 생성**

```sql
-- 이 쿼리가 느리다면
SELECT * FROM users WHERE email = 'test@example.com';

-- 이렇게 인덱스 생성
CREATE INDEX idx_users_email ON users(email);
```

- **복합 인덱스는 순서가 중요**

```sql
-- WHERE customer_id = ? AND status = ? ORDER BY created_at
CREATE INDEX idx_orders_cust_status_created
ON orders(customer_id, status, created_at);
```

예시)

```sql
-- 느린 쿼리
SELECT * FROM products 
WHERE category_id = 5 
  AND price BETWEEN 1000 AND 5000 
  AND is_active = 1
ORDER BY created_at DESC
LIMIT 20;
```

```sql
-- 등호 조건(category_id, is_active) → 범위 조건(price) 순서
CREATE INDEX idx_products_search 
ON products(category_id, is_active, price);
```

**주의:** 범위 조건 이후의 컬럼은 인덱스를 타지 않으므로, `ORDER BY created_at`은 이 인덱스로 최적화되지 않는다.

- **인덱스가 타지 않는 흔한 실수**

```sql
-- ❌ 컬럼에 함수 사용
WHERE YEAR(created_at) = 2024

-- ✅ 범위로 변경
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'

-- ❌ 암묵적 타입 변환
WHERE phone = 01012345678  -- phone이 VARCHAR인 경우

-- ✅ 타입 일치
WHERE phone = '01012345678'

-- ❌ LIKE 앞쪽 와일드카드
WHERE name LIKE '%홍길동'

-- ✅ 뒤쪽 와일드카드만
WHERE name LIKE '홍길동%'
```

### Cardinality

그룹 내 요소의 개수를 의미한다. 인덱스를 적용할 때 카디널리티가 높은(중복 수치가 낮은) 컬럼에 대해 적용해주면 된다.

카디널리티는 PK 컬럼이 높을 텐데, 유니크한 특성으로 인해 중복되는 값이 존재하지 않으므로 수치가 높을 것이다.

### 인덱스를 적용하기 좋은 추가 Case

1. Cardinality
2. 조건절에 자주 사용되는 컬럼인 경우

   Where, Join, OrderBy에 자주 사용되는 컬럼인 경우 인덱스를 적용하는게 좋다.

3. insert, update, delete가 자주 발생하지 않는 컬럼인 경우

   B+Tree구조는 삽입, 수정, 삭제 성능 대신 조회의 성능에 최적화된 구조이다. 이러한 연산이 자주 발생하지 않는 컬럼에 대해 인덱스를 만드는 것이 좋다.

4. 규모가 작지 않은 테이블

### 인덱스 유의사항

방치되는 인덱스는 삭제하자. 방치되는 인덱스라면 성능저하의 요인이다.

반대로 자주 사용되는 칼럼이더라도 삽입, 수정, 삭제가 자주 일어나는지 고려해야 한다.

궁금한 점 : 복잡한 Join, Where문 등이 걸려있는 테이블/컬럼이 있다. → 만약 Read도 많이 일어나는데, Create/Update/Delete도 많이 발생된다면?
