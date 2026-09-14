---
title: "Thread Safe"
date: 2026-02-22 00:43:00 +0900
categories: [spring, async-thread]
tags: [thread-safe, jpa, concurrency]
---

### Thread Safe 정의

Thread Safe란 여러 스레드가 동시에 같은 코드나 데이터에 접근해도 프로그램이 올바르게 동작하는 것을 의미한다.

Race Condition, 데이터 손상, 예측 불가능한 결과가 발생하지 않아야 한다.

### JPA Entity는 Thread Safe하지 않다.

**1. 엔티티는 단순한 POJO이다.**

내부 동기화 매커니즘이 없는 mutable(변하기 쉬운/값이 변하는) 객체이다.

여러 스레드가 동시에 setter를 호출하면 데이터가 깨질 수 있다.

**2. EntityManager가 thread-safe하지 않다.**

JPA 스펙에서 EntityManager는 명시적으로 Thread-Safe하지 않다고 정의한다.

→ 하나의 EntityManager(Persistence Context)는 하나의 스레드에서만 사용해야 한다.

같은 엔티티 인스턴스가 여러 스레드의 Persistence Context에서 관리되면 Dirty Checking, flush 등에서 문제가 발생한다.

**3. 실무에서 안전한 이유**

Spring에서는 트랜잭션 단위로 EntityManager가 ThreadLocal에 바인딩되므로 일반적인 사용 패턴에서는 자연스럽게 스레드별로 분리된다.

문제가 되는 경우 : 엔티티 인스턴스를 스레드 간 공유, static필드에 캐싱, CompletableFuture 등으로 다른 스레드에 넘길 때

```java
// 위험: 엔티티를 다른 스레드에 넘김
MyEntity entity = entityManager.find(MyEntity.class, 1L);
CompletableFuture.runAsync(() -> {
    entity.setName("new name"); // 다른 스레드에서 수정 → 문제 발생 가능
});

// 안전: DTO로 변환 후 넘김
MyDto dto = new MyDto(entity.getId(), entity.getName());
CompletableFuture.runAsync(() -> {
    // dto는 불변이거나 독립적인 객체이므로 안전
});
```
