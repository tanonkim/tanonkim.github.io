---
title: "HikariCP DeadLock 벗어나기 정리"
date: 2026-02-01 16:28:00 +0900
categories: [database, connection-pool]
tags: [hikaricp, deadlock, jpa]
---

[HikariCP Dead lock에서 벗어나기 (이론편) \| 우아한형제들 기술블로그](https://techblog.woowahan.com/2664/)

[HikariCP Dead lock에서 벗어나기 (실전편) \| 우아한형제들 기술블로그](https://techblog.woowahan.com/2663/)

### 장애 배경

우아한형제들의 메시지 플랫폼에서 장애가 발생했는데, 원인을 분석해보니 세 가지 항목 간의 상관관계가 있었다.

- Message Queue Consumer Thread 갯수
- HikariCP Maximum Pool Size
- @GeneratedValue 전략 (SEQUENCE, AUTO)

세 요소의 관계를 정확히 이해하지 못한 채 설정 → 부하 상황에서 Thread간 Connection을 차지하기 위한 Race Condition이 발생했고, HikariCP에서 DeadLock이 발생

### HikariCP Connection 획득/반납 매커니즘

- **HikariCP 내부구조**

HikariCP는 내부적으로 ConcurrentBag이라는 구조체를 사용하여 Connection을 관리한다. Connection을 얻을 때는 `HikariPool.getConnection()` → `ConcurrentBag.borrow()` 메서드를 통해 사용가능한 Connection을 리턴받는다.는다.

- **getConnection() 3단계 로직**

Connection을 요청하면 HikariCP는 다음 3단계를 순차적으로 수행한다.

1. 1단계 ThreadLocal 확인 (이전 방문 내역 확인)

   해당 Thread가 이전에 사용했던 Connection이 있는지 확인

   있다면 그 Connection을 우선 반환 (성능 최적화 목적)

2. 전체 Pool에서 Idle Connection 탐색

   ThreadLocal에 없으면 전체 Pool을 스캔하여 사용가능한 Connection 탐색

   다른 Thread가 사용하지 않는 idle connection을 steal할 수 있음

3. handleOffQueue에서 대기

   사용가능한 Connection이 없으면 handleOffQueue에 등록하여 다른 Thread가 Connection을 반납할 때까지 대기

   Connection Timeout 시간(기본 30초)까지 대기 후, 여전히 없으면 `SQLTransientConnectionException` 발생

- **Connection 반납 과정**

Transaction이 commit되거나 rollback되면 `connection.close()`가 호출되고, 이때 `ConcurrentBag.requite()`이 실행되어 Connection이 Pool에 반납됩니다. 반납된 Connection은 handOffQueue에서 대기 중인 다른 Thread에게 전달됩니다.

### Deadlock 발생 시나리오

- **장애환경 조건**

```java
CPU Core: 4개
Thread Count: 16개
HikariCP MaximumPoolSize: 10개
하나의 Task에서 동시에 요구되는 Connection 갯수: 2개
```

- **왜 Connection이 2개인가**

```java
@Entity
class Message {
    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private long id;
    private String title;
    private String contents;
}

@Transactional
public Message save(final Message message) {
    return repository.save(message);
}
```

하나의 Connection으로 Insert될 것 같지만, 실제로 2개의 Connection이 필요하다. → `@GeneratedValue(strategy = GenerationType.AUTO)`

Spring Boot 2.x에서 Hibernate는 ID 생성을 위해 `hibernate_sequence` 테이블을 별로도 조회/업데이트 한다. 해당 작업은 별도의 Connection에서 수행된다.

1. **Connection 1**: Root Transaction용 (실제 Insert 쿼리)
2. **Connection 2**: ID 생성을 위한 Sequence 조회/업데이트

- **DeadLock 발생 과정**

조건을 다시 보면:

Thread 16개, Pool Size 10개, 각 Thread가 Connection 2개 필요

1. 부하 상황에서 16개의 Thread가 거의 동시에 Connection을 요청
2. 10개의 Thread만 첫 번째 Connection을 획득하고 Transaction 시작
3. 이 10개의 Thread가 ID 생성을 위해 **두 번째 Connection**을 요청
4. 하지만 Pool에는 idle Connection이 이미 없음! (10개 모두 사용 중)
5. 10개의 Thread 모두 handOffQueue에서 대기
6. **교착 상태 발생**: 모든 Thread가 서로가 Connection을 반납하기를 기다림
7. 아무도 Transaction을 완료할 수 없으므로, 누구도 Connection을 반납하지 못함

결과적으로 Connection Timeout(30초)이 지나면 `SQLTransientConnectionException`이 발생하고, Transaction이 rollback되면서 Connection이 반납된다. 그제야 대기 중인 다른 Thread가 Connection을 받아 간헐적으로 1~2건씩 성공하는 현상이 나타난다.

### 해결방법

- Pool Size 공식

  HikariCP에서는 Dead lock을 피하기 위한 최소 Pool Size 공식을 제시한다.

  ```
  pool size = Tn × (Cm - 1) + 1

  Tn : 전체 Thread 수
  Cm : 하나의 Task에서 동시에 필요한 최대 Connection 수

  pool size = 16 × (2 - 1) + 1 = 17
  ```

- 해결책
  - **Pool Size 조정**: 위 공식에 맞게 Maximum Pool Size 증가
  - **Dead lock 가능성 테스트 방법**:
    - HikariCP의 Maximum Pool Size를 **1로 설정**한 뒤 1건씩 Query 실행
    - 정상 실행되지 않고 Connection timeout이 발생하면 Dead lock 가능성이 있는 코드
  - **Nested Transaction 사용 자제**:
    - 보이지 않는 Dead lock을 유발할 수 있음
    - 특히 `@GeneratedValue` 전략 선택 시 주의
  - **ID 생성 전략 변경 검토**:
    - `GenerationType.IDENTITY` 사용 시 별도 Connection 불필요 (DB가 자동 증가)
    - 단, 배치 Insert 성능에는 영향 있을 수 있음

### 핵심 교훈

1. Maximum Pool Size는 "많으면 좋다"가 아님: Thread 수와 동시에 필요한 Connection 수의 관계를 이해하고 설정해야 함
2. ORM의 숨겨진 동작 이해 필요: `@GeneratedValue(AUTO/SEQUENCE)`가 별도 Connection을 사용한다는 사실을 인지해야 함
3. 부하 테스트 필수: 평상시에는 문제없다가 부하 상황에서만 Dead lock이 발생할 수 있음
4. Connection Pool 내부 동작 이해: HikariCP의 3단계 getConnection 로직과 handOffQueue 동작 방식을 이해하면 문제 진단이 수월해짐
