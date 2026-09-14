---
title: "Spring Transaction"
date: 2026-02-22 16:50:00 +0900
categories: [spring, core]
tags: [spring, transaction]
---

## 1. 트랜잭션 시작 시 일어나는 일

### 전체 흐름

client → AOP Proxy → TransactionInterceptor → PlatformTransactionManager → DataSource → DB Connection

### 단계별 상세 과정

1. AOP 프록시 진입

   Spring은 @Transactional이 선언된 Bean에 대해 프록시 객체를 생성한다. 기본적으로 CGLIB프록시(클래스 기반)를 사용하며, 인터페이스가 있는 경우 JDK Dynamic Proxy를 사용할 수도 있다. 해당 메서드를 호출하면 실제 객체가 아닌 프록시 객체의 메서드가 먼저 실행된다.

2. TransactionInterceptor 동작

   프록시 내부에서 TransactionInterceptor가 동작한다. 이 인터셉터는 @Transactional 어노테이션에 정의된 속성(propagation, isolation, timeout, readOnly, rollbackFor 등)을 읽어 TransactionAttribute 객체로 변환한다.

3. PlatformTransactionManager에 트랜잭션 시작 요청

   TransactionInterceptor는 PlatformTransactionManager.getTransaction(TransactionDefiniton)을 호출한다. 이 때 **전파 속성(Propagation)**에 따라 분기된다.

   기존 트랜잭션이 있는 경우 : 새로운 트랜잭션 생성

   기존 트랜잭션이 없는 경우 : 전파 속성에 따라 기존 트랜잭션에 참여, 새 트랜잭션 생성, 예외 던지기

4. DataSource에서 Connection 획득

   새 트랜잭션을 시작해야 하는 경우, DataSource.getConnection()을 통해 DB커넥션을 획득한다. 커넥션 풀(HikariCP)에서 사용 가능한 커넥션을 가져온다.

   해당 시점이 실제로 DB 자원을 점유하기 시작하는 시점이다.

5. Auto-Commit 비활성화

   획득한 커넥션의 setAutoCommit(false)를 호출한다. JDBC의 기본값은 auto-commit이 true이므로, 트랜잭션을 관리하려면 반드시 꺼야 한다.

6. 트랜잭션 동기화(TransactionSynchronizationManager)

   커넥션을 TransactionSynchronizationManger에 바인딩한다. 이 매니저는 **ThreadLocal 기반**으로 동작하며 현재 스레드에 다음의 정보를 저장한다.

   | 저장항목 | 설명 |
   | --- | --- |
   | Connection | 현재 트랜잭션이 사용하는 DB 커넥션 |
   | TransactionStatus | 트랜잭션의 상태 정보(신규 여부, 롤백 마킹 등) |
   | TransactionSynchronization | 커밋/롤백 시 콜백 목록 |
   | 격리수준, readOnly 등 | 트랜잭션 속성 들 |

7. 비즈니스 로직 실행

   실제 @Transactional 메서드의 본문이 실행된다.

8. 커밋 또는 롤백

   - 정상 종료 : connection.commit() → setAutoCommit(true) → 커넥션 풀에 반환
   - 예외 발생 : connection.rollback() → setAutoCommit(true) → 커넥션 풀에 반환

### **유의사항**

프록시 기반이므로 this.method()같은 내부 호출에서는 트랜잭션이 적용되지 않는다.

ThreadLocal기반이므로 다른 스레드에서는 동일 트랜잭션에 참여할 수 없다.

커넥션 획득 시점 = 트랜잭션 시작 시점이며, 이때부터 커넥션 풀의 자원을 점유한다.

## 2. 트랜잭션이 길어진다면

실무에서 많은 장애를 유발하는 포인트 중 하나.

### 커넥션 풀 고갈

가장 치명적인 문제이다.

트랜잭션이 시작되면 커넥션을 점유하고, 커밋/롤백 될 때까지 반환하지 않는다. 트랜잭션이 길어지면 커넥션 반환이 지연되고, 다른 요청들이 커넥션을 받지 못해 대기하게 된다.

```java
[시나리오]
HikariCP max pool size : 10
긴 트랜잭션 10개가 동시에 실행 (generateType 등등)
11번째 요청 -> 커넥션 획득 대기 -> timeout -> 장애 
```

HikariCp의 기본 ConnectionTimeout은 30초이다. 이 시간 내에 커넥션을 얻지 못하면 SQLTransientConnectionException이 발생한다.

### DB Lock 장시간 보유

트랜잭션 내에서 UPDATE, DELETE, SELECT … FOR UPDATE 등을 수행하면 해당 row에 lock이 걸린다. 트랜잭션이 길어지면 lock이 오래 유지되어 다른 트랜잭션이 해당 row에 접근하지 못하고 대기하게 된다.

**연쇄효과:**

1. 트랜잭션 A가 row를 lock한 채 외부 API 호출 (3초 소요)
2. 트랜잭션 B,C,D가 같은 row를 변경하려고 대기
3. 대기중인 **트랜잭션들도 각각 커넥션을 점유**
4. 커넥션 풀 고갈 + Lock 대기 + DeadLock 가능성 증가

### Undo Log 및 MVCC 비용 증가

MVCC : Multi Version Concurrency Control

InnoDB(Mysql)의 경우 긴 트랜잭션은 undo log를 오래 유지해야 한다. 다른 트랜잭션들이 MVCC를 통해 이전 버전의 데이터를 읽어야 하기 때문이다.

- undo log가 쌓이면서 디스크 사용량 증가
- 오래된 스냅샷을 유지 → purge 스레드 지연
- 극단적인 경우 DB 성능 전체가 저하

### 메모리 압박(JPA/Hibernate 환경)

JPA의 영속성 컨텍스트(1차 캐시)는 트랜잭션 범위 내에서 유지된다. 긴 트랜잭션에서 대량의 엔티티를 로드하면 영속성 컨텍스트가 커진다.

- flush() 시 dirty checking 대상이 많아져 성능 저하
- 메모리 사용량 증가 (OOM 위험)

### Timeout으로 인한 예상치 못한 롤백

@Transactional(timeout = 5) 또는 글로벌 타임아웃 설정이 있을 때, 긴 트랜잭션은 중간에 타임아웃으로 강제 롤백될 수 있다. 이미 처리된 비즈니스 로직(외부 API 호출, 메세지 발송 등)은 되돌릴 수 없어서 데이터 정합성 문제가 발생한다.

### 실무 장애 사례 패턴

| 패턴 | 설명 | 결과 |
| --- | --- | --- |
| 트랜잭션 내 외부 API 호출 | 외부 API 응답 지연 시 커넥션 점유 장기화 | 커넥션 풀 고갈 |
| 대량 배치를 단일 트랜잭션으로 | 수만 건을 한 트랜잭션에서 처리 | OOM, Lock 경합 |
| **N+1 쿼리 + 긴 트랜잭션** | 쿼리 수가 폭발적으로 증가 | DB 부하 + 응답 지연 |
| 트랜잭션 내 파일 업로드/다운로드 | I/O 대기 시간만큼 커넥션 점유 | 커넥션 풀 고갈 |

## 3. 안티패턴

### 1. 트랜잭션 내에서 외부 API 호출

**코드 예시**

```java
// 안티패턴
@Transactional
public void createOrder(OrderRequest request) {
    Order order = orderRepository.save(new Order(request));
    
    // 외부 API 호출 — 여기서 3초 지연되면 커넥션도 3초 점유
    paymentGateway.requestPayment(order.getPaymentInfo());
    
    order.markAsPaid();
    orderRepository.save(order);
}
```

```java
// 개선: 트랜잭션 분리
public void createOrder(OrderRequest request) {
    Order order = saveOrder(request);  // 트랜잭션 1
    
    PaymentResult result = paymentGateway.requestPayment(order.getPaymentInfo());  // 트랜잭션 밖
    
    updateOrderPaymentStatus(order.getId(), result);  // 트랜잭션 2
}

@Transactional
public Order saveOrder(OrderRequest request) {
    return orderRepository.save(new Order(request));
}

@Transactional
public void updateOrderPaymentStatus(Long orderId, PaymentResult result) {
    Order order = orderRepository.findById(orderId).orElseThrow();
    order.updatePaymentStatus(result);
}
```

### 2. 불필요하게 큰 트랜잭션 범위

**코드 예시**

```java
// 안티패턴: 서비스 메서드 전체를 하나의 트랜잭션으로
@Transactional
public void processMonthlyReport() {
    List<User> users = userRepository.findAll();           // 10만 건
    List<ReportData> data = calculateReportData(users);    // 무거운 연산 (5초)
    reportRepository.saveAll(generateReports(data));       // DB 저장
    emailService.sendReports(users);                       // 이메일 발송 (외부 I/O)
    slackNotifier.notify("Report completed");              // Slack 알림
}
```

```java
// 개선: 트랜잭션 범위 최소화
public void processMonthlyReport() {
    List<User> users = userRepository.findAll();  // readOnly 트랜잭션 또는 별도 조회
    List<ReportData> data = calculateReportData(users);  // 순수 연산, 트랜잭션 불필요
    
    saveReports(generateReports(data));  // 트랜잭션은 DB 저장에만
    
    emailService.sendReports(users);     // 트랜잭션 밖
    slackNotifier.notify("Report completed");
}

@Transactional
public void saveReports(List<Report> reports) {
    reportRepository.saveAll(reports);
}`
```

### 3. 내부 호출

**코드 예시**

```java
// 안티패턴: 같은 클래스 내부에서 @Transactional 메서드 호출 -> 별도 클래스로 분리
@Service
public class OrderService {
    
    public void processOrder(Long orderId) {
        // this.updateStock()은 프록시를 거치지 않으므로 트랜잭션이 적용되지 않음
        updateStock(orderId);
    }
    
    @Transactional
    public void updateStock(Long orderId) {
        // 트랜잭션이 걸려야 하지만 실제로는 걸리지 않는다!
        stockRepository.decrease(orderId);
    }
}
```

### 4. **무분별한 @Transactional(readOnly = true) 남용**

**코드 예시**

```java
// 안티패턴: 읽기 전용인데 쓰기 작업이 포함됨
@Transactional(readOnly = true)
public UserDto getUser(Long id) {
    User user = userRepository.findById(id).orElseThrow();
    user.updateLastAccessTime(LocalDateTime.now());  // 변경이 flush되지 않아 무시됨!
    return UserDto.from(user);
}
```

### 5. 예외를 삼켜버리는 패턴

**코드 예시 - 다시보기**

```java
// 안티패턴: catch 내에서 예외를 삼키면 롤백이 안 됨
@Transactional
public void transferMoney(Long fromId, Long toId, BigDecimal amount) {
    try {
        accountRepository.withdraw(fromId, amount);
        accountRepository.deposit(toId, amount);
    } catch (Exception e) {
        log.error("이체 실패", e);
        // 예외를 삼켰으므로 Spring은 정상 완료로 판단 → 커밋됨
        // withdraw만 반영되고 deposit은 안 된 상태로 커밋
    }
}
```

### 6. 트랜잭션 내에서 대량 데이터 처리

**코드 예시**

```java
// 안티패턴
@Transactional
public void migrateAllUsers() {
    List<User> users = userRepository.findAll();  // 100만 건 전부 로드
    for (User user : users) {
        user.migrate();
    }
    // flush 시 100만 건 dirty checking → 극심한 지연
}
```

```java
// 개선: 청크 단위 처리
public void migrateAllUsers() {
    int page = 0;
    int size = 500;
    Page<User> chunk;
    
    do {
        chunk = migrateChunk(page, size);  // 각 청크가 별도 트랜잭션
        page++;
    } while (chunk.hasNext());
}

@Transactional
public Page<User> migrateChunk(int page, int size) {
    Page<User> users = userRepository.findAll(PageRequest.of(page, size));
    users.forEach(User::migrate);
    return users;
}
```

### **7. Checked Exception에 대한 롤백 미처리**

**코드 예시**

```java
// 안티패턴: checked exception이면 롤백이 안 됨
@Transactional
public void process() throws BusinessException {
    repository.save(entity);
    if (someCondition) {
        throw new BusinessException("실패");  // RuntimeException이 아니면 롤백 안 됨!
    }
}
```

```java
// 개선: rollbackFor 명시
@Transactional(rollbackFor = BusinessException.class)
public void process() throws BusinessException {
    // ...
}
```

## 4. 전파속성

### 4-1. 전파 속성 전체 정리

| 전파 속성 | 기존 트랜잭션 있을 때 | 기존 트랜잭션 없을 때 |
| --- | --- | --- |
| **`REQUIRED`**` (기본값)` | 기존 트랜잭션에 참여 | 새 트랜잭션 생성 |
| **`REQUIRES_NEW`** | 기존 트랜잭션 일시 중단, 새 트랜잭션 생성 | 새 트랜잭션 생성 |
| **SUPPORTS** | 기존 트랜잭션에 참여 | 트랜잭션 없이 실행 |
| **NOT_SUPPORTED** | 기존 트랜잭션 일시 중단, 트랜잭션 없이 실행 | 트랜잭션 없이 실행 |
| **MANDATORY** | 기존 트랜잭션에 참여 | **예외 발생** (IllegalTransactionStateException) |
| **NEVER** | **예외 발생** (IllegalTransactionStateException) | 트랜잭션 없이 실행 |
| **NESTED** | 중첩 트랜잭션 생성 (Savepoint) | 새 트랜잭션 생성 |

### REQUIRED

대부분의 비즈니스 로직에 사용한다. 호출하는 쪽과 같은 트랜잭션에 참여하여, 하나의 논리적 작업 단위로 묶인다.

### **REQUIRES_NEW**

기존 트랜잭션과 **독립적으로** 커밋/롤백되어야 하는 경우에 사용한다.

### **SUPPORTS**

트랜잭션이 있으면 참여하고, 없으면 비트랜잭션으로 실행한다.

주로 **읽기 전용 메서드**에서, 호출자의 트랜잭션 컨텍스트를 유연하게 따르고 싶을 때 사용한다.

### **NOT_SUPPORTED**

트랜잭션이 불필요하거나, 기존 트랜잭션 안에서 실행되면 안 되는 작업에 사용한다.

트랜잭션을 일시 중단시키므로 **커넥션을 반환**하고, 작업 완료 후 기존 트랜잭션이 재개된다.

### **MANDATORY**

반드시 기존 트랜잭션 안에서 호출되어야 하는 메서드에 사용한다. **실수로 트랜잭션 없이 호출되는 것을 방지**하는 방어적 용도이다.

### **NEVER**

트랜잭션이 있는 상태에서 호출되면 예외를 던진다. 트랜잭션 컨텍스트에서 절대 실행되면 안 되는 작업에 사용한다.

### **NESTED**

부모 트랜잭션 내에서 **Savepoint를 생성**하여 중첩 트랜잭션을 만든다.

중첩 트랜잭션이 롤백되어도 부모 트랜잭션은 계속 진행할 수 있다. 단, 부모가 롤백되면 중첩도 함께 롤백된다.

## 5. REQUIRES_NEW

### **1. 동작 원리**

`REQUIRES_NEW`는 기존 트랜잭션을 일시중단하고, 완전히 새로운 트랜잭션을 시작한다.

새 트랜잭션은 **새로운 DB 커넥션**을 사용한다.

예)

```java
[Thread-1]
기존 트랜잭션 (Connection A) — suspend
    └→ 새 트랜잭션 (Connection B) — 실행 → commit/rollback
기존 트랜잭션 (Connection A) — resume
```

→ 커넥션을 2개 점유한다. 중첩이 깊어지면 커넥션 풀이 빠르게 고갈될 수 있다.

### **2. 사용하면 좋은 경우**

1. 감사로그

   메인 비즈니스 로직이 실패하더라도 "시도했다"는 기록은 남겨야 하는 경우.

2. **독립적으로 커밋되어야 하는 부분 작업**

   예) 알림 발송 실패가 전체 트랜잭션을 롤백시키면 안되는 경우

   ```java
   @Transactional
   public void completePayment(Long paymentId) {
       Payment payment = paymentRepository.findById(paymentId).orElseThrow();
       payment.complete();
       
       try {
           notificationService.saveNotification(payment.getUserId(), "결제 완료");
           // REQUIRES_NEW — 알림 저장 실패가 결제 완료를 롤백시키면 안 됨
       } catch (Exception e) {
           log.warn("알림 저장 실패", e);
       }
   }
   ```

3. 시퀀스

   고유번호를 발급하는 로직이 메인 트랜잭션 롤백 시에도 번호가 유지되어야 하는 경우

   ```java
   @Transactional(propagation = Propagation.REQUIRES_NEW)
   public String generateOrderNumber() {
       Sequence seq = sequenceRepository.findByName("ORDER");
       seq.increment();
       return seq.getCurrentValue();
       // 메인 트랜잭션이 롤백되어도 번호는 소모된 것으로 유지
   }
   ```

### **3. REQUIRES_NEW 사용 시 주의사항**

| 주의사항 | 설명 |
| --- | --- |
| 커넥션 2개 점유 | 기존 + 새 커넥션을 동시에 사용. **남용 시 커넥션 풀 고갈** |
| 데드락 위험 | 기존 트랜잭션이 lock한 row를 새 트랜잭션이 접근하면 데드락 |
| 롤백 독립성 | 새 트랜잭션 커밋 후 기존 트랜잭션이 롤백되면 새 트랜잭션은 **되돌릴 수 없음** |
| Self-Invocation 불가 | 같은 클래스 내부 호출 시 프록시를 거치지 않아 **전파 속성이 무시됨** |

## 6. 트랜잭션 경계 설계

### 1. 계층별 트랜잭션 배치 비교

| 계층 | 트랜잭션 배치 | 장점 | 단점 |
| --- | --- | --- | --- |
| Controller | `@Transactional` on Controller | 없음 | 요청 전체가 트랜잭션, 커넥션 장기 점유, 뷰 렌더링까지 포함 |
| **Service** | `@Transactional` on Service (✅ 권장) | **비즈니스 단위로 트랜잭션 관리, 계층 분리 명확** | 서비스 내 외부 호출 포함 시 설계 주의 필요 |
| Repository | `@Transactional` on Repository | 개별 쿼리 단위 관리 | 여러 쿼리를 하나로 묶을 수 없음 |

### 2. Service 계층에 트랜잭션 배치

원칙 : 트랜잭션은 Service 계층에 두되, 비즈니스 로직을 DB작업과 외부 작업으로 분리한다.

```java
Controller (트랜잭션 없음)
    └→ Facade/Orchestrator Service (트랜잭션 없음, 흐름 제어만)
        ├→ Domain Service (트랜잭션 있음, 순수 DB 작업)
        ├→ External API 호출 (트랜잭션 밖)
        └→ Domain Service (트랜잭션 있음, 결과 반영)
```

### 3. 잘못된 설계 시 발생하는 사이드 이펙트

- Ctontroller 내의 @Transactional

  **코드 예시**

  ```java
  // Controller에 트랜잭션
  @RestController
  public class OrderController {
      
      @Transactional
      @PostMapping("/orders")
      public ResponseEntity<OrderResponse> createOrder(@RequestBody OrderRequest request) {
          Order order = orderService.createOrder(request);
          // JSON 직렬화까지 트랜잭션 내에서 실행
          // → 커넥션 점유 시간 불필요하게 증가
          // → Lazy Loading 문제를 숨겨 나중에 더 큰 장애 발생
          return ResponseEntity.ok(OrderResponse.from(order));
      }
  }
  ```

  1. 뷰 렌더링/JSON 직렬화 시간까지 커넥션 점유
  2. Lazy Loading이 트랜잭션 안에서 동작 → N+1문제를 은폐
  3. 부하 시 커넥션 풀 고갈

- 외부 API를 트랜잭션 안에서 호출

  외부 API 장애 시 커넥션이 묶임 → 연쇄 장애

  예) 결제 API timeout이 커넥션 timeout보다 길면 커넥션 풀 고갈

  결제 성공했는데 이후 DB 저장에서 실패 시 → 결제는 됐지만 주문은 안 된 상태

- Repository에서 트랜잭션을 둘 때

  여러 DB 작업의 원자성이 보장되지 않음

  부분 실패 시 데이터 정합성 깨짐

## 7. 롤백 매커니즘

### 1. 기본 롤백 규칙

| 예외 타입 | 기본 동작 | 커스텀 |
| --- | --- | --- |
| **RuntimeException** (Unchecked) | 롤백 | `noRollbackFor`로 커밋 가능 |
| **Error** | 롤백 | `noRollbackFor`로 커밋 가능 |
| **Checked Exception** | 커밋 | `rollbackFor`로 롤백 가능 |

```java
// 롤백 제어 예시
@Transactional(
    rollbackFor = {BusinessException.class, IOException.class},
    noRollbackFor = {NotificationFailedException.class}
)
public void process() { /* ... */ }
```

### 2. 동기 환경에서의 롤백

**REQUIRED**

하나의 물리적 트랜잭션 안에서 모든 참여 메서드가 실행된다. 어느 한 곳에서 예외가 발생하면 전체가 롤백된다. (내부 메서드의 예외를 외부에서 catch해도 이미 rollback-only로 마킹 → 커밋할 수 없음)

```java
@Transactional  // 물리 트랜잭션 시작
public void outerMethod() {
    repository.save(entityA);     // 성공
    innerService.doSomething();   // 여기서 예외 발생 → 전체 롤백
    repository.save(entityB);     // 실행 안 됨
}

// InnerService
@Transactional  // REQUIRED — 기존 트랜잭션에 참여
public void doSomething() {
    repository.save(entityC);
    throw new RuntimeException("실패");
    // → 트랜잭션에 rollback-only 마킹
    // → outerMethod에서 catch하더라도 커밋 시 UnexpectedRollbackException 발생
}
```

**REQUIRES_NEW**

각각 독립된 물리 트랜잭션이므로 서로의 롤백에 영향을 주지 않는다.

```java
@Transactional
public void outerMethod() {
    repository.save(entityA);  // 외부 트랜잭션
    
    try {
        innerService.doSomething();  // REQUIRES_NEW — 별도 트랜잭션
        // innerService가 예외 → 내부 트랜잭션만 롤백
    } catch (Exception e) {
        log.warn("내부 트랜잭션 실패, 외부는 계속 진행");
    }
    
    repository.save(entityB);  // 외부 트랜잭션 계속 진행 → 커밋
}
```

**NESTED**

SavePoint를 활용한다. 중첩 트랜잭션이 롤백되면 SavePoint까지만 롤백되고, 부모 트랜잭션은 계속 진행할 수 있다.

```java
@Transactional
public void outerMethod() {
    repository.save(entityA);
    
    try {
        innerService.doSomething();  // NESTED — Savepoint 생성
    } catch (Exception e) {
        // Savepoint까지만 롤백, entityA는 유지
    }
    
    repository.save(entityB);  // 커밋 가능
}
```

### 3. 비동기 환경에서의 롤백

비동기 메서드는 다른 스레드에서 실행되므로, 호출자의 트랜잭션에 참여할 수 없다. 이유 : TransactionSynchronizationManager가 ThreadLocal기반이기 떄문이다. 비동기 메서드의 `@Transactional`은 완전히 독립된 새 트랜잭션을 생성한다.

**전략 1 : 이벤트 기반 + TransactionalEventListener**

호출자의 트랜잭션이 **커밋된 후에만** 비동기 작업을 실행한다.

```java
@Service
public class OrderService {
    private final ApplicationEventPublisher eventPublisher;
    
    @Transactional
    public void createOrder(OrderRequest request) {
        Order order = orderRepository.save(new Order(request));
        
        // 트랜잭션이 커밋된 후에만 이벤트 발행
        eventPublisher.publishEvent(new OrderCreatedEvent(order.getId()));
        
        // 여기서 예외 발생 → 롤백 → 이벤트 리스너 실행 안 됨
    }
}

@Component
public class OrderEventListener {
    
    @Async
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handleOrderCreated(OrderCreatedEvent event) {
        // 주문 트랜잭션이 커밋된 후에만 실행됨
        // 이 안에서 실패해도 주문에 영향 없음
        notificationService.sendNotification(event.getOrderId());
    }
}
```

**전략 2: 보상 트랜잭션**

비동기 작업 실패 시 이전 작업을 되돌리는 보상 로직을 구현한다.

```java
@Service
public class SagaOrchestrator {
    
    public void processOrder(OrderRequest request) {
        Long orderId = null;
        try {
            orderId = orderService.createOrder(request);          // Step 1
            paymentService.processPayment(orderId);                // Step 2 (비동기)
            inventoryService.reserveStock(orderId);                // Step 3 (비동기)
        } catch (Exception e) {
            // 보상 트랜잭션: 역순으로 되돌리기
            if (orderId != null) {
                inventoryService.cancelReservation(orderId);       // Step 3 보상
                paymentService.refund(orderId);                    // Step 2 보상
                orderService.cancelOrder(orderId);                 // Step 1 보상
            }
        }
    }
}
```

**전략 3: Outbox Pattern**

트랜잭션 안에서 이벤트를 Outbox 테이블에 저장하고, 별도 프로세스가 Outbox를 폴링하여 비동기 작업을 수행한다.

```java
@Transactional
public void createOrder(OrderRequest request) {
    Order order = orderRepository.save(new Order(request));
    
    // 같은 트랜잭션 안에서 이벤트를 Outbox 테이블에 저장
    outboxRepository.save(new OutboxEvent(
        "ORDER_CREATED", 
        objectMapper.writeValueAsString(new OrderCreatedPayload(order.getId()))
    ));
    // 트랜잭션 롤백 시 outbox 이벤트도 함께 롤백 → 정합성 보장
}

// 별도 스케줄러가 Outbox를 폴링하여 처리
@Scheduled(fixedDelay = 1000)
@Transactional
public void processOutbox() {
    List<OutboxEvent> events = outboxRepository.findUnprocessed();
    for (OutboxEvent event : events) {
        try {
            messageBroker.publish(event);
            event.markAsProcessed();
        } catch (Exception e) {
            event.incrementRetryCount();
        }
    }
}
```
