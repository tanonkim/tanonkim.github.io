---
title: "GeneratedType 키 생성 전략"
date: 2026-02-08 17:20:00 +0900
categories: [jpa, entity]
tags: [jpa, generated-value, primary-key]
---

### Id의 정의

JPA에서 Entity객체를 정의할 때 @Id 속성을 함께 정의해야 한다. 이 Id를 정의할 때 직접 할당하는 방법과 자동생성하는 방법이 있다.

1. 직접 할당하는 방법
   1. @Id 어노테이션만으로 id를 지정한다.

      ```java
      @Entity
      public class Member {
          @Id
          private String id; // 애플리케이션에서 직접 세팅

          // ...
      }

      // 사용
      Member member = new Member();
      member.setId("MEMBER_001");
      em.persist(member);
      ```

2. 자동생성
   1. @Id와 @GeneratedValue를 같이 사용한다.
   2. GenerationType 옵션으로 전략을 지정한다.

### Generation Type 종류

- TABLE
- IDENTITY
- SEQUENCE
- AUTO

### **TABLE**

특정 벤더에 의존적이지 않다. Sequence 테이블을 생성해서 데이터베이스 시퀀스를 흉내낼 Id를 할당한다. jpa ddl auto 설정이 되어있지 않다면 시퀀스 테이블 생성이 선행되어야 한다.

```java
@Entity
@TableGenerator(
    name = "member_table_generator",
    table = "MY_SEQUENCES",        // 시퀀스 역할을 할 테이블명
    pkColumnValue = "MEMBER_SEQ",
    allocationSize = 50
)
public class Member {
    @Id
    @GeneratedValue(strategy = GenerationType.TABLE, generator = "member_table_generator")
    private Long id;
}
```

시퀀스를 테이블로 흉내내는 것으로 별도 테이블에 대한 lock이 발생한다. 모든 DB에 사용가능하다는 장점이 있지만, 성능이 가장 떨어진다.

### **IDENTITY**

기본키 생성을 DB에게 위임한다. (MySQL의 경우는 AUTO INCREMENT)

특정 DB 벤더에 의존적이다.Id가 null일 경우 해당 객체의 Id를 DB의 AUTO_INCREMENT를 가져와 할당한다.

```java
@Entity
public class Member {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
}
```

- 객체를 저장할 때 발생하는 일

  JPA에서 영속성 컨텍스트에서 객체를 관리하기 위해서 무조건 ID속성(PK값이 있어야 한다. 이유 : JPA 입장에서 Map의 Key 값이 없다면 객체의 값을 넣을 수 있는 방법이 없기 때문이다.)

  AUTO_INCREMENT의 경우 애플리케이션에서 해당 값이 얼마인지 모르기 때문에 DB에 insert를 날려야 ID를 알 수 있다.

  →  save() 메서드를 호출하는 시점에 insert 쿼리 / flush가 된다, 이렇게 할당한 객체는 영속성 컨텍스트에 1차 캐시에 값을 넣는다.

- batch insert

  JPA는 보통 트랜잭션 커밋 시점에 쓰기 지연 SQL 저장소를 **한번에 flush**한다.

  그런데 IDENTITY는 persist()시점에 즉시 insert를 날려야 하기에 쓰기 지연(batch inser)가 불가하다. 대량 insert 성능이 중요한 경우 SEQUENCE 전략이 유리

### **SEQUENCE**

DB의 sequence 객체를 통해 유일한 값을 순서대로 생성한다. (특정 DB 벤더에 의존적이다.)

이 전략은 sequence를 사용하는 Oracle, DB2, H2 등의 DB에서 사용하며, @SequenceGenerator(시퀀스를 생성하는 어노테이션)과 함께 사용할 수 있다.

만약 Sequence 객체를 사용하지 않는 DB 벤더를 이용할 경우 Sequence를 관리할 객체 테이블을 생성한다. (jpa ddl-auto 설정이 필수)

```java
@Entity
@SequenceGenerator(
    name = "member_seq_generator",
    sequenceName = "member_seq",  // 실제 DB 시퀀스 이름
    initialValue = 1,
    allocationSize = 50           // 성능 최적화 핵심
)
public class Member {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "member_seq_generator")
    private Long id;
}
```

- 객체를 저장할 때 발생하는 일

  sequence 값만 시퀀스 객체에서 조회해오면 되기에, 엔티티 테이블에 해당 객체를 바로 insert 할 필요가 없다.

- allocationSize

  기본값은 50이다. DB에서 시퀀스 값을 호출할 때 50만큼 미리 확보하고, 49개는 메모리에서 순차적으로 할당 → 50개 엔티티를 저장할 때 시퀀스 조회를 위한 네트워크 호출이 1번만 발생한다.

### **AUTO**

DB 벤더에 따라 3가지 전략 중 하나를 선택한다.

DB 벤더를 변경해도 전략을 수정하지 않아도 되지만, SEQUENCE나 TABLE 전략을 선택한다면 sequence나 키 테이블을 생성해두어야 함을 주의해야 한다. jpa ddl-auto를 사용하면 hibernate가 알아서 시퀀스 테이블을 생성해준다.

```java
@Entity
public class Member {
    @Id
    @GeneratedValue(strategy = GenerationType.AUTO) // 또는 그냥 @GeneratedValue
    private Long id;
}
```

주의점 : HikariCP DeadLock에 주의하자

하나의 Connection으로 Insert될 것 같지만, 실제로 2개의 Connection이 필요하다. → `@GeneratedValue(strategy = GenerationType.AUTO)`
