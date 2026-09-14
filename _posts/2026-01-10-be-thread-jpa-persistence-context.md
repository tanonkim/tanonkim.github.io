---
title: "JPA 영속성 컨텍스트"
date: 2026-01-10 14:16:00 +0900
categories: [jpa, persistence-context]
tags: [jpa, persistence-context]
---

### 영속성 컨텍스트(Persistence Context)

영속성 컨텍스트 : 애플리케이션과 데이터베이스 사이에 위치, 엔티티를 영구 저장(영속)하는 가상의 데이터베이스이다. 논리적인 개념으로 JPA에서 엔티티의 생명주기를 관리하고, 데이터베이스와의 상호작용을 최적화하는 역할을 한다.

### EntityManager, EntityManagerFactory

EntityManager는 영속성 컨텍스트에 접근할 수 있는 주요 인터페이스이다. 예를 들어 entityManager는 Persist를 사용해서 엔티티를 영속성 컨텍스트에 저장할 수 있다.

```java
EntityManager.persist(entity)
```

EntityManager는 DB 테이블과 매핑된 엔티티에 대해 CRUD 작업을 수행할 수 있는 메서드를 제공한다. 그리고 엔티티의 생명주기를 관리한다.

**EntityManager는 `Thread Safe` 하지 않기**에 각 요청마다 새로운 EntityManager 인스턴스를 생성하여 사용한다. 이를 관리하는 것이 **`EntityManagerFactory`** 인터페이스이다. EntityManagerFactory는 Thread Safe로 구현되어 여러 스레드에서 동시에 접근해도 문제가 없다.

![](/assets/img/2026-01-10/be-thread-jpa-persistence-context-1.png)

### 1차 캐시, 쓰기 지연 저장소

영속성 컨텍스트는 1차 캐시 영역과 쓰기지연 저장소로 구성되어있다.

![](/assets/img/2026-01-10/be-thread-jpa-persistence-context-2.png)

1차 캐시 영역은 Map 형태(key-value)로 엔티티를 저장한다. 여기서 Key는 식별자이며 value는 엔티티 객체이다.

1차 캐시에 저장할 수 있는 값은 식별자를 통한 쿼리만 해당한다. 식별자는 데이터베이스의 PK를 사용하여 저장하는데, PK가 아닌 다른 값을 통해 쿼리문을 실행하면 영속성 컨텍스트에 저장되지 않는다.

쓰기 지연 저장소에서 SQL을 바로 실행하지 않고, 영속성 컨텍스트의 명령에 따라 SQL을 모아두었다가 한 번에 데이터베이스에 전송한다. 이를 통해 트랜잭션 관리와 성능 최적화가 가능하다.

### JPA의 이점

JPA는 영속성 컨텍스트를 통해 아래의 이점을 제공한다.

1. 1차 캐시 조회
2. 동일성 보장
3. 트랜잭션을 지원하는 쓰기 지연
4. 변경감지

^

1. **1차 캐시 영역 조회**

EntityManager가 엔티티를 조회하면 먼저 1차 캐시 영역에서 조회를 시도한다.

```java
Member findMember = entityManager.find(Memeber.class, 1L);
```

1차 캐시에 해당 엔티티가 있는 경우 1차 캐시에서 엔티티를 찾아 반환한다.

1차 캐시에서 엔티티가 없는 경우 Select 쿼리를 실행해 데이터를 조회하고, 데이터가 존재하면 1차 캐시에 저장하고 엔티티 객체를 반환한다.

![](/assets/img/2026-01-10/be-thread-jpa-persistence-context-3.png)

그러나 트랜잭션 종료 시 영속성 컨텍스트가 비워지기 때문에 1차 캐시가 성능에 미치는 영향은 크지 않을 수 있다.

1. **동일성 보장**

영속성 컨텍스트는 동일한 PK값을 가진 엔티티를 조회할 때 항상 같은 객체를 반환하여 동일성을 보장한다. 예를 들어 PK값이 같은 데이터를 두 번 조회해도 동일한 객체가 반환된다.

```java
Member member1 = em.find(Member.class, 1L);
Member member2 = em.find(Member.class, 1L);

member1 == member2; // true
```

동일한 객체라고 판단하는 이유 : 1차 캐시에서 이미 조회된 동일한 엔티티가 존재하면 새로운 객체를 생성하지 않고 1차 캐시에 저장된 엔티티를 그대로 반환한다. → 이로써 영속성 컨텍스트는 동일성을 보장하며, 같은 트랜잭션 내에서 PK가 동일한 엔티티는 항상 같은 인스턴스를 사용한다.

1. 트랜잭션을 지원하는 쓰기 지연

트랜잭션 내에서 발생하는 모든 데이터 변경 작업은 즉시 데이터베이스에 반영되지 않고 쓰기 지연 저장소에 보관된다. 변경사항은 EntityManager.flush() 메서드가 호출될 때 한꺼번에 데이터베이스에 반영된다.

![](/assets/img/2026-01-10/be-thread-jpa-persistence-context-4.png)

`flush` : 영속성 컨텍스트를 DB와 동기화하는 작업

**Flush가 발생하는 경우**

1. 트랜잭션이 커밋될 때

   commit()메서드가 호출되기 직전에 flush()가 자동으로 호출된다.

2. 명시적으로 flush()를 호출할 때

   EntityManager.flush()를 호출하면 flush()가 발생한다.

3. JPQL 쿼리 실행 시

   JPQL이나, SQL 쿼리가 실행되지 전에 영속성 컨텍스트와 데이터베이스의 상태를 일치시키기 위해 flush가 발생된다.

flush가 완료된 후 commit()메서드가 호출되면 트랜잭션이 종료되며, DB는 영구적으로 변경사항을 반영한다. 트랜잭션 중에 예외가 발생하여 롤백되면 flush된 변경사항들도 취소된다.

flush : 쓰기지연저장소에 있는 SQL문을 데이터베이스에 반영, 트랜잭션을 종료하지 않음 (롤백된다면 flush된 변경사항도 취소됨)

commit : 트랜잭션을 종료하며, flush된 변경사항을 데이터베이스에 영구적으로 반영함

1. **변경감지**

변경감지는 flush가 발생할 때 이루어진다.

- 변경감지 절차
  1. 변경감지 : 영속성 컨텍스트는 엔티티의 변경사항을 감지한다.
  2. 쓰기 지연 저장소에 쿼리 저장 : 수정된 엔티티에 대한 업데이트 쿼리가 생성되어 쓰기 지연 저장소에 저장된다.
  3. DB 동기화 : 쓰기 지연 저장소에 저장된 쿼리문들이 데이터베이스에 반영된다.

- 변경감지 이루어지는 과정
  1. 1차 캐시에 저장된 엔티티의 초기 상태를 스냅샷으로 저장한다.
  2. 이후 개발자가 엔티티의 필드를 변경하면 영속성 컨텍스트는 현재 엔티티의 상태를 스냅샷과 비교하여 변경된 부분을 감지한다.

  이렇게 변경사항을 찾아내는 과정을 Dirty Checking(변경감지)라고 한다.

- `@Transactional`

  `@Transactional` 을 사용하면 save없이도 변경사항을 저장할 수 있다. 스프링에서 제공하는 @Transactional이 적용되는 메서드에는 트랜잭션이 commit(종료)되기 직전에 flush가 자동으로 되어 변경감지가 이루어진다. (save를 호출하지 않아도 자동으로 DB에 반영된다.)
