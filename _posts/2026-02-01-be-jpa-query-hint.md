---
title: "JPA QueryHint"
date: 2026-02-01 19:53:00 +0900
categories: [jpa, persistence-context]
tags: [jpa, query-hint, hibernate]
---

### QueryHint

JPA에서 제공하는 @QueryHint는 쿼리 실행 시 Hibernate나 JPA 구현체에게 특정 힌트를 전달하여 동작을 제어하는 기능이다.

사용예시)

- Dirty Checking을 생략해서 flush 방지
- 쿼리 결과를 2차 캐시에 저장
- JDBC 쿼리를 타임 아웃에 적용

```java
@QueryHints({
	@QueryHint(name = "org.hibernate.readOnly", value = "true")
})
@Query("SELECT m FROM Member m")
List<Member> findAllReadOnly();
```

### 자주 사용하는 Hint

- org.hibernate.readOnly
  - 조회한 엔티티를 dirty checking 대상에서 제외
  - → 불필요한 스냅샷 비교, flush 방지 → 성능 향상
  - 주의 : 조회 전용으로만 사용할 것, 변경 시 반영이 안됨

- org.hibernate.cacheable
  - 쿼리 결과를 **`2차 캐시`**에 저장하도록 설정
  - → 동일 쿼리 재사용 시 DB 접근 없이 캐시에서 가져옴
  - 전제조건 : 2차 캐시 설정 및 캐시 전략이 필요(예) Redis ? 등)

- javax.persistence.query.timeout
  - JDBC 쿼리 실행 최대 시간을 지정
  - → 일정 시간 초과 시 예외 발생 : 리스크 제어 관리, 장애 회피

- org.hibernate.comment
  - SQL에 주석을 삽입해 로그 분석 시 어떤 로그인지 추적
  - JPA 쿼리가 많을 때 원인 찾는데 도움이 됨

- org.hibernate.fetchSize
  - JDBC 드라이버에 전달되는 fetch size 힌트
  - → 대용량 데이터 조회 시 커서 기반으로 성능 튜닝 가능

참고 : Hibernate Hint는 EntityManager로 만든 쿼리에도 적용할 수 있다.

### 보조수단인 Hint

QueryHint는 “무조건적인 최적화 도구”가 아니다. 마이크로 최적화이다.

대부분의 서비스에서 성능 병목은 **다른 데서 생긴다**

- JVM GC의 현대 성능은 예전과 다르다.

  예전의 Hibernate의 Dirty Checking도 비용이 있었고, **readOnly** 설정만으로도 눈에 띄는 차이가 있었다. 하지만 지금은 상황이 다르다.

  | 항목 | 과거 | 현재 |
  | --- | --- | --- |
  | GC 비용 | 빈번한 stop-the-world | G1GC, ZGC로 매우 안정적 |
  | Dirty Checking | 수천 건도 체감 차이 났음 | 대부분 무시할 수준 |
  | 스냅샷 비교 | 메모리·CPU 소비 이슈 | 대부분 negligible(무시 가능) |

- 언제 사용해야 하냐

  → 아주 구체적인 상황에서 쓸만하다.

  1. 대용량 조회 + 엔티티 변경 불가 조건이 명확한 경우
  2. flush 주기를 최대한 미루고 싶은 경우
  3. 2차 캐시를 반드시 활성화해야 하는 고정 쿼리
  4. JDBC 드라이버 레벨에서 타임아웃을 제어해야 할 때

### 성능 개선이 필요할 때

QueryHint는 Micro Tuning(마이크로 최적화)이다. 성능 개선이 필요할 때 아래를 먼저 확인해야 한다.

| 항목 | 설명 |
| --- | --- |
| 인덱스 설계 | 적절한 composite index 없이 빠른 쿼리는 없다 |
| Fetch Join | N+1을 없애고 쿼리 수 자체를 줄임 |
| EntityGraph | 필요한 필드만 가져와 메모리 낭비 방지 |
| Native Query | 복잡한 join, group, window 함수 등은 JPQL보다 효율적 |
| 쿼리 구조 개선 | where절 재구성, limit 사용, 존재 체크 분리 등 |
