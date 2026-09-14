---
title: "MySQL & Batch update 시 조심할 것"
date: 2026-03-09 12:49:00 +0900
categories: [database, mysql]
tags: [mysql, batch, pagination]
---

> 실제 서비스명, 도메인, 리소스 식별자 등은 보안을 위해 예시 값으로 바꿨습니다.
{: .prompt-info }

## 읽기 배치

### 기본 원칙

읽기 작업 배치는 반드시 Reader 인스턴스에서 실행

신규 배치 개발 시 적절한 **인덱스** 유무 확인 하기

### 페이징 쿼리 방식 비교

**OFFSET ❌**

```java
SELECT coupon_member_id, type_no, member_no 
	FROM coupon_member 
	ORDER BY coupon_member_id LIMIT 0, 1000;

SELECT coupon_member_id, type_no, member_no F
	ROM coupon_member 
	ORDER BY coupon_member_id LIMIT 1000, 1000;
```

구현은 단순하지만, **성능 저하**

조회 범위가 뒤로 갈 수록 불필요한 데이터까지 읽음

**CURSOR ✅**

```java
SELECT coupon_member_id, type_no, member_no 
	FROM coupon_member 
	ORDER BY coupon_member_id LIMIT 1000;

SELECT coupon_member_id, type_no, member_no 
	FROM coupon_member 
	WHERE coupon_member_id > "첫번째페이지의 마지막ID값" 
	ORDER BY coupon_member_id LIMIT 1000;
```

primary key 등 정렬 컬럼을 이용해 읽기 시작점을 지정

항상 일정한 범위만 읽어서 성능이 균일함

batch뿐만 아니라, 일반 조회에도 동일하게 적용이 권장된다.

## 쓰기 배치

0.1초 sleep을 주자

### Insert

1000건 미만 청크 단위로 작업

각 청크마다 0.1초 sleep후 다음 청크 처리

### Delete & Update

사전에 DBA와 반드시 쿼리 검수 후 작업

적절한 인덱스 유무 확인

500건 미만 청크 단위로 작업

각 청크마다 0.1초 sleep 후 다음 청크 처리

⚠️ **500건 이상 작업 시 table lock 발생** 주의
