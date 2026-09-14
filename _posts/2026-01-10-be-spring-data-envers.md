---
title: "Spring Data Envers를 이용한 엔티티 변경 이력 관리"
date: 2026-01-10 15:51:00 +0900
categories: [spring, core]
tags: [spring, jpa, envers]
---

가 있다. JPA를 사용중이라면 spring-data-envers를 통해 이를 편리하게 설정하고 관리할 수 있다.

spring-data-envers는 hibernate-envers의 wrapping 프로젝트로 envers를 편리하게 사용할 수 있는 기능(RevisionRepository, 메타데이터 조회)을 제공한다.

1. Dependency

   ```java
   implementation("org.springframework.data:spring-data-envers
   ```

2. EnableJpaRepositories 추가

   ```java
   @EnableJpaRepositories(repositoryFactoryBeanClass = EnversRevisionRepositoryFactoryBean.class)
   @SpringBootApplicationpublic class Application { 	
   	
   	public static void main(String[] args) {	
   			SpringApplication.run(Application.class, args);	
   	} 
   }
   ```

3. Auditing 활성화

   검사할 엔티티에 @Audited를 추가한다.

   ```java
   @Getter
   @Entity
   @Audited
   public class User extends BaseEntity {
       @Id
       @GeneratedValue(strategy = GenerationType.IDENTITY)
       private Long id;

       private String name;

       private String email;
   }
   ```

전체 테이블에 대해서 이력을 남기고 싶다면 클래스에 `@Audited`를 붙이고, 정해진 필드만 관리하고 싶다면 각 필드에 대해 `@Audited` 어노테이션을 붙이면 된다. 또 전체 필드 중 제외하고 싶은 필드가 있다면 `@NotAudited`를 붙이면 감사 대상에서 제외된다.

이력 조회 대상에 대해 변경되었는지 여부를 flag를 통해 관리할 수도 있다.

이 flag 를 조회해서 해당 버전에서 어떤 엔티티가 바뀌었는지 boolean 값으로 관리할 수 있다.

```java
@Audited(withModifiedFlag = true, modifiedColumnName = "nameChanged")
private String name;
```

중요한 점 :  Transaction 단위로 revision을 관리한다는 것

application.yaml

```java
spring:
  jpa:
    properties.org.hibernate.envers:
        audit_table_suffix: _history
        revision_field_name: revision_id
        revision_type_field_name: revision_type
        # 해당 transaction 변경이력 있는 데이터 불러오기
        # track_entities_changed_in_revision: true
```
