---
title: "@Async, 쓰레드와 트랜잭션 관계"
date: 2026-02-15 19:10:00 +0900
categories: [spring, async-thread]
tags: [spring, async, transaction]
---

### JPA에서 스레드와 트랜잭션 관계

원칙 : 트랜잭션은 스레드에 바인딩된다.

Spring의 트랜잭션 관리는 ThreadLocal 기반으로 동작한다.

### 1. TransactionSynchronizationManager의 역할

Spring의 `TransactionSynchronizationManager`에서 ThreadLocal 변수들을 통해 현재 스레들의 트랜잭션 상태를 관리한다.

```java
public abstract class TransactionSynchronizationManager {
    private static final ThreadLocal<Map<Object, Object>> resources = new NamedThreadLocal<>("Transactional resources");
    private static final ThreadLocal<Boolean> actualTransactionActive = new NamedThreadLocal<>("Actual transaction active");
    // ...
}
```

**이 말은 아래와 같다.**

- 다른 스레드에서는 이 리소스들에 대해 접근할 수 없다.
- DB Connection역시 현재 스레드에 바인딩된다.
- EntityManager(영속성 컨텍스트)도 현재 스레드에 바인딩된다.

### 2. @Transactional과 스레드의 관계

```java
@Transactional
public void parentMethod() {
    // 현재 스레드 A에 트랜잭션 T1, EntityManager EM1, Connection C1이 바인딩됨

    // ✅ 같은 스레드에서 호출 → 동일 트랜잭션 참여 (REQUIRED 기준)
    childService.doSomething();

    // ❌ 새 스레드 → 부모 트랜잭션과 완전히 독립
    new Thread(() -> {
        // 여기서는 T1, EM1, C1에 접근 불가
        // 트랜잭션이 필요하면 새로 시작해야 함
        anotherService.doAsync();
    }).start();
}
```

### **3. @Async와 트랜잭션**

```java
@Service
public class OrderService {

    @Transactional
    public void createOrder(Order order) {
        orderRepository.save(order);     // 트랜잭션 A (스레드 1)
        notificationService.sendAsync(order); // 새 스레드로 실행됨
    }
}

```

```java
@Service
public class NotificationService {

    @Async
    @Transactional  // ← 별도의 새 트랜잭션이 시작됨 (스레드 2)
    public void sendAsync(Order order) {
        // 부모의 트랜잭션과 무관
        // 부모가 롤백돼도 여기는 이미 커밋될 수 있음
    }
}
```

### **4. 영속성 컨텍스트(EntityManager)와 스레드**

| 상황 | EntityManager | 트랜잭션 |
| --- | --- | --- |
| 같은 스레드, `REQUIRED` | 공유 | 공유 |
| 같은 스레드, `REQUIRES_NEW` | 새로 생성 | 새로 생성 |
| 다른 스레드 (`@Async`, `CompletableFuture` 등) | 새로 생성 | 새로 생성(있다면) |
| 트랜잭션 없는 스레드 | 없음 | 없음 |

### **5. 실무에서 자주 발생하는 문제들**

1. **자식 스레드에서 Lazy 로딩 실패**

```java
@Transactional
public void process(Long userId) {
    User user = userRepository.findById(userId).get(); // 프록시 로딩

    CompletableFuture.runAsync(() -> {
        user.getOrders().size(); // ❌ LazyInitializationException
        // 영속성 컨텍스트가 이 스레드에 없음
    });
}
```

해결 : 자식 스레드에 넘기기 전에 필요한 데이터를 미리 fetch하거나 ID만 넘기고 자식 스레드에서 새로 조회

```java
// 해결 1: 미리 초기화
Hibernate.initialize(user.getOrders());

// 해결 2: ID만 전달하고 새로 조회
CompletableFuture.runAsync(() -> {
    User freshUser = userRepository.findById(userId).get();
    freshUser.getOrders().size(); // ✅ 새 영속성 컨텍스트에서 조회
});
```

1. 자식 스레드 결과를 부모 트랜잭션에 반영

```java
@Transactional
public void process() { // 스레드 A
    // 자식 스레드의 작업이 부모 트랜잭션에 포함되지 않음
    CompletableFuture<Result> future = CompletableFuture.supplyAsync(() -> { // 스레드 B
        return heavyComputation(); // 이건 별도 트랜잭션
    });

    Result result = future.join();
    // result를 저장하는 것은 부모 트랜잭션에서 가능하지만,
    // 자식 스레드에서 DB에 쓴 내용은 부모의 롤백과 무관
}
```

1. OSIV와 스레드

OSIV(Open Session In View)가 켜져 있어도, OSIV가 바인딩한 EntityManager는 원래 요청 스레드에만 유효합니다. `@Async`나 별도 스레드에서는 OSIV의 혜택을 받지 못한다.

```java
스레드 1 (HTTP 요청)          스레드 2 (@Async)
┌─────────────────────┐      ┌─────────────────────┐
│ ThreadLocal         │      │ ThreadLocal         │
│  ├─ Transaction A   │      │  ├─ Transaction B   │
│  ├─ EntityManager A │      │  ├─ EntityManager B │
│  └─ Connection A    │      │  └─ Connection B    │
└─────────────────────┘      └─────────────────────┘
서로 완전히 독립 — 롤백/커밋 전파 없음
```

**핵심 규칙:**

1. **1 스레드 = 1 트랜잭션 컨텍스트** (ThreadLocal 기반)
2. 스레드가 달라지면 트랜잭션, EntityManager, Connection 모두 독립
3. 자식 스레드에는 엔티티가 아닌 **ID나 DTO**를 넘기는 것이 안전
4. 멀티스레드 환경에서 트랜잭션 일관성이 필요하면 **보상 트랜잭션(`Saga`)** 패턴 등을 고려

트랜잭션이 행동하는게 있어서 ConnectionPool과의 관계는?
