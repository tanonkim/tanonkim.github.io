---
title: "JPA AuditorAware"
date: 2026-02-15 19:28:00 +0900
categories: [jpa, entity]
tags: [jpa, auditing]
---

> 실제 서비스명, 도메인, 리소스 식별자 등은 보안을 위해 예시 값으로 바꿨습니다.
{: .prompt-info }

### 1. AuditorAware란?

JPA에는 엔티티가 **생성되거나 수정될 때** 자동으로 "누가" 그 작업을 했는지 기록하는 기능이 있다. `AuditorAware<T>`는 그 "누가"에 해당하는 값을 제공하는 인터페이스이다.

Spring Data JPA의 **Auditing 기능**의 한 축이며, 나머지 축은 "언제"를 담당하는 `@CreatedDate` / `@LastModifiedDate` 이다.

### 2. 전체 Auditing 구조

Auditing은 4가지 어노테이션으로 구성된다.

| 어노테이션 | 역할 | 출처 |
| --- | --- | --- |
| @CreatedDate | 생성시각 | Spring제공 |
| @LastModifiedDate | 수정시각 | Spring제공 |
| @CreatedBy | **생성한사람** | AuditAware 제공 |
| @LastModifiedBy | **수정한사람** | AuditAware 제공 |

@CreatedBy, @LastModifiedBy은 직접 알려줘야 한다. → 그 역할이 AuditorAware이다.

### 동작 흐름

1. 엔티티 persist/update 발생
2. Spring JPA Auditing 인터셉서 동작
   1. @CreatedDate / @LastModifiedDate → 현재 시각 주입
   2. @CreatedBy, @LastModifiedBy
3. AuditorAware.getCurrentAuditor() 호출
4. 반환된 값을 엔티티 필드에 주입

### 활성화 조건

1. `@EnableJpaAuditing` 설정

```java
@EnableJpaAuditing
@EnableFeignClients(basePackages = "com.example.domain")
@ConfigurationPropertiesScan(basePackages = "com.example.config.properties")
@SpringBootApplication
public class ExampleApplication {

    public static void main(String[] args) {
        SpringApplication.run(ExampleApplication.class, args);
    }
}
```

1. EntityListeners 적용

```java
@Getter
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class) // 해당 리스너가 실제로 값을 주입
public abstract class BaseTimeEntity {

    @CreatedDate
    @Column(name = "create_at", nullable = false, updatable = false)
    @Comment("등록일")
    private LocalDateTime createAt;

    @LastModifiedDate
    @Column(name = "update_at", nullable = false)
    @Comment("수정일")
    private LocalDateTime updateAt;
}
```

```java
@Getter
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class) // 해당 리스너가 실제로 값을 주입
public abstract class BaseTimeEntity {

    @CreatedDate
    @Column(name = "create_at", nullable = false, updatable = false)
    @Comment("등록일")
    private LocalDateTime createAt;

    @LastModifiedDate
    @Column(name = "update_at", nullable = false)
    @Comment("수정일")
    private LocalDateTime updateAt;
}
```

AudiitingEntityListener가 JPA 라이프사이클 이벤트(@Prepersist, @PreUpdate)를 감지해서 그 시점에 AuditorAware.getCurrentAuditor()를 호출한다.

```java
@Slf4j
@Component
public class AuditorAwareImpl implements AuditorAware {

  @Override
  public Optional getCurrentAuditor() {
    Long loginMemberOAuthId = 0L;

    if (AuthenticationUtils.getLoginMemberId() != null) {
      loginMemberOAuthId = AuthenticationUtils.getLoginMemberId();
    }

    return Optional.ofNullable(loginMemberOAuthId);
  }
}
```

### 호출 시점 정리

| 상황 | 호출되는 시점 | 채워지는 필드 |
| --- | --- | --- |
| entityManager.persist(entity) | @PrePersist | @CreatedBy + @LastModifiedBy |
| entity.setName("변경") → flush | @PreUpdate | @LastModifiedBy만 |

중요한 점은 **최초 저장 시 `@LastModifiedBy`도 함께 채워진다**는 것이다.

→ 데이터는 어디에서 채워질까? with ThreadLocal

context holder에 유저정보, ThreadLocal로 구현되어있다.

### 주의할 점

1. Spring SecurityContext가 없는 상황 고려

   스케줄러, 이벤트 리스너, @Async 메서드를 통해 엔티티를 저장하면 SecurityContextHolder가 비어있다. 코드에서는 0L로 처리

2. @Transactional(readOnly = true)

   읽기모드에서는 호출이 되지 않는다. Dirty Checking이 일어나지 않으므로 @PreUpdate가 발생하지 않는다.

3. 테스트

   Auditing을 끄거나, Mock해야 할 수 있다. SecurityContext가 없는 환경에서 NPE가 발생할 수 있다. → NPE가 발생할 수 있으므로 `@MockBean AuditorAware`로 처리하거나 `@WithMockUser`를 사용
