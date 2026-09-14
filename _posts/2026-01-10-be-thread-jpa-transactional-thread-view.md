---
title: "스레드 관점에서 보는 JPA + @Transacional"
date: 2026-01-10 17:23:00 +0900
categories: [spring, async-thread]
tags: [jpa, transaction, thread]
---

## JPA 영속성 컨텍스트와 직결되는 쓰레드

1. **EntityManager는 쓰레드 세이프하지 않다**

- EntityManager는 단일 쓰레드에서만 사용해야 함
- 여러 쓰레드가 하나의 EntityManager를 공유하면 데이터 정합성 문제 발생

1. **트랜잭션과 쓰레드의 관계**

- Spring의 `@Transactional`은 **ThreadLocal**을 사용해 트랜잭션을 쓰레드에 바인딩
- 다른 쓰레드로 넘어가면 트랜잭션 컨텍스트가 전파되지 않음

1. **지연 로딩(Lazy Loading) 문제**

- 비동기 처리(`@Async`, CompletableFuture)에서 Lazy 엔티티 접근 시 `LazyInitializationException` 발생
- 영속성 컨텍스트가 다른 쓰레드에 없기 때문
- OSIV

### **왜 EntityManager가 Thread Safe하지 않아야 할까?**

```java
┌─────────────────────────────────────────────────────────┐
│                  EntityManagerFactory                   │
│                   (Thread Safe ✅)                      │
│                   애플리케이션당 1개                        │
└──────────────┬────────────────────┬─────────────────────┘
               │                    │
               ▼                    ▼
┌──────────────────────┐  ┌──────────────────────┐
│   Thread A           │  │   Thread B           │
│  ┌────────────────┐  │  │  ┌────────────────┐  │
│  │ EntityManager  │  │  │  │ EntityManager  │  │
│  │ (Thread Safe ❌)│ │  │  │ (Thread Safe ❌)│  │
│  └────────────────┘  │  │  └────────────────┘  │
│  ┌────────────────┐  │  │  ┌────────────────┐  │
│  │ 영속성 컨텍스트 │  │  │  │  │ 영속성 컨텍스트    │  │
│  │ - 1차 캐시    │  │  │  │  │ - 1차 캐시      │  │
│  │ - 쓰기지연저장소│  │  │  │  │ - 쓰기지연저장소.  │  │
│  └────────────────┘  │  │  └────────────────┘  │
└──────────────────────┘  └──────────────────────┘
```

가정 : 만약 여러 쓰레드가 하나의 EntityManager를 공유한다면?

```java
EntityManager em = emf.createEntityManager();  // 하나만 생성

// Thread A
Member memberA = em.find(Member.class, 1L);  // 1차 캐시에 저장
memberA.setName("변경A");

// Thread B (동시 실행)
Member memberB = em.find(Member.class, 1L);  // 같은 1차 캐시 접근!
memberB.setName("변경B");
```

데이터 정합성이 붕괴된다. 누구의 변경이 반영이 될 지 모르며, 1차 캐시의 동일성 보장이 깨진다. + 쓰기 지연 저장소의 SQL의 순서가 꼬인다.

### @Transactional과 쓰레드

**ThreadLocal**

- ThreadLocal의 기본 개념

```java
ThreadLocal<String> threadLocal = new ThreadLocal<>();

// 각 쓰레드마다 독립적인 값을 가짐
threadLocal.set("Thread A의 데이터");  // Thread A에서 실행
threadLocal.get();  // Thread A에서만 "Thread A의 데이터" 반환
```

ThreadLocal은 쓰레드별로 독립된 저장 공간을 제공한다. 다른 쓰레드에서 접근할 수 없다.

**@Transactional과 쓰레드**

Spring의 @Transactional은 ThreadLocal을 사용해 트랜잭션을 현재 쓰레드에 바인딩한다.

HTTP요청 → Thread Pool에서 Thread 할당

```
        ┌───────────────────────────────────┐
        │         ThreadLocal 저장소          │
        │  ┌─────────────────────────────┐  │
        │  │ TransactionSynchronization  │  │
        │  │ Manager (트랜잭션 정보)        │  │
        │  └─────────────────────────────┘  │
        │  ┌─────────────────────────────┐  │
        │  │ EntityManagerHolder         │  │
        │  │ (현재 EntityManager)         │  │
        │  └─────────────────────────────┘  │
        └───────────────────────────────────┘
```

**트랜잭션 전파(Propagation)가 같은 쓰레드에서만 동작하는 이유:**

ThreadLocal에 접근이 불가하기 때문이다.

```java
@Service
public class OrderService {
    
    @Transactional
    public void createOrder() {
        // 트랜잭션 시작 → ThreadLocal에 저장
        
        memberService.updatePoint();  // 같은 쓰레드 → 트랜잭션 전파 ✅
        
        // 이건 어떨까?
        CompletableFuture.runAsync(() -> {
            // 다른 쓰레드! → ThreadLocal 접근 불가 ❌ (runAsync는 별도의 쓰레드 풀에서 작업을 함)
            // 트랜잭션이 없는 상태로 실행됨
            logService.saveLog();
        });
    }
}
```

**격리수준(Isolation)과 동시성**

격리수준 :  서로 **다른 쓰레드**(**트랜잭션**)가 동시에 같은 데이터에 접근할 때 어떻게 처리할지를 정의한다.

```java
Thread A (트랜잭션 A)          Thread B (트랜잭션 B)
        │                            │
        ▼                            ▼
  SELECT * FROM                SELECT * FROM
  member WHERE id=1            member WHERE id=1
```

READ_UNCOMMITTED : B가 A의 커밋 전 데이터를 볼 수 있음 (Dirty Read)

READ_COMMITTED : B는 A가 커밋한 이후에 변경 데이터 봄

REPEATABLE_READ : B가 읽은 데이터는 A가 수정 불가

SERIALIZABLE : 완전 직렬화 (동시 실행 x)

**예) 재고 차감 문제**

```java
@Transactional(isolation = Isolation.READ_COMMITTED)  // 기본값
public void decreaseStock(Long productId) {
    Product product = productRepository.findById(productId);
    
    // Thread A: stock = 10 읽음
    // Thread B: stock = 10 읽음 (동시에!)
    
    product.decreaseStock(1);
    
    // Thread A: stock = 9로 저장
    // Thread B: stock = 9로 저장 (10-1이니까)
    // 결과: 2개 차감되어야 하는데 1개만 차감(상황 : 두 개의 스레드가 각각 1개씩 차감을 요청)
}

// 해결: 비관적 락 또는 낙관적 락 사용
@Lock(LockModeType.PESSIMISTIC_WRITE)
Product findByIdForUpdate(Long id);
```

### **Lazy Loading**

문제 : @Async에서 LazyInitializationException

```java
@Service
public class OrderService {
    
    @Transactional
    public void processOrder(Long orderId) {
        Order order = orderRepository.findById(orderId).get();
        // 영속성 컨텍스트에 order 저장됨
        
        asyncService.sendNotification(order);  // 비동기 호출
    }
}

@Service
public class AsyncService {
    
    @Async
    public void sendNotification(Order order) {
        // 🔴 다른 쓰레드에서 실행됨
        // 🔴 영속성 컨텍스트가 없음
        
        String memberName = order.getMember().getName();  
        // LazyInitializationException 발생
    }
}
```

해결방법

1. 필요한 데이터를 미리 로딩

```java

@Transactional
public void processOrder(Long orderId) {
    Order order = orderRepository.findById(orderId).get();
    String memberName = order.getMember().getName();  // 여기서 로딩
    
    asyncService.sendNotification(order.getId(), memberName);  // 값만 전달
}

```

방법 2: @Async 메서드에서 새 트랜잭션 시작

```java
@Async
@Transactional  // 새로운 트랜잭션 (새로운 영속성 컨텍스트)
public void sendNotification(Long orderId) {
    Order order = orderRepository.findById(orderId).get();  // 새로 조회
    // ...
}
```

**OSIV와 쓰레드**

OSIV : Open Session In View (영속성 컨텍스트를 View(JSP 등)까지 열어둠)

```java
┌─────────────────────────────────────────────────────────────┐
│                      HTTP 요청 쓰레드                          │
│                                                             │
│  [Filter/Interceptor] → [Controller] → [Service] → [View]   │
│         │                                           │       │
│         └────    영속성 컨텍스트 생존 범위 (OSIV ON)   ───┘       │
│                                                             │
│                    [Service]만 트랜잭션                        │
│                         │                                   │
│         └─ 영속성 컨텍스트 생존 범위 (OSIV OFF) ─┘                │
└─────────────────────────────────────────────────────────────┘
```

OSIV가 켜져 있으면 **같은 쓰레드 내**에서 View까지 Lazy Loading이 가능하지만, 커넥션을 오래 점유하는 단점이 있다.
