---
title: "JPA 1차/2차 캐시"
date: 2026-02-01 20:11:00 +0900
categories: [jpa, persistence-context]
tags: [jpa, cache, hibernate]
---

### 캐시

Cache : 나중의 요청에  대한 결과를 미리 저장했다가 빠르게 사용하는 것

![](/assets/img/2026-02-01/be-jpa-first-second-level-cache-1.png)

위로 갈수록 빠르고 비싸고, 밑으로 갈수록 느리고 저렴한 저장소이다. Secondary Memory는 디스크 영역이다.

기본적으로 데이터는 컴퓨터가 꺼져도 저장이 되어야 하기 떄문에 Secondary Memory에 저장이 되는데, Main Memory/Cache Memory/CPU Registers등에 저장하여 더 빠르고 쉽게 접근할 수 있는 방법이 생겼다.

캐시를 사용하면 MySQL등 관계형 DB에 접근하는 것보다 빠른데, 이유는 메모리 접근이 디스크 접근보다 빠르기 때문이다.

⇒ Database보다 더 빠른 Memory에 더 자주 접근하고, 덜 자주 바뀌는 데이터를 저장할 때 적합하다.

### 1차 캐시

**1차 캐시 : 영속성 컨텍스트 내부에 엔티티를 보관하는 저장소**

예) Java → Hibernate 1차 캐시 (Persistance Context)

네트워크를 통해 DB에 접근하는 시간 비용 > 애플리케이션 서버에서 내부 메모리에 접근하는 시간 비용

따라서 조회한 데이터를 메모리에 캐시해서 DB 접근 횟수를 줄이면 애플리케이션 성능을 개선할 수 있다.

![](/assets/img/2026-02-01/be-jpa-first-second-level-cache-2.png)

- 일반적인 웹 애플리케이션 환경은 트랜잭션이 시작하고 종료할 때 까지만 1차 캐시가 유효하다.
- OSIV를 사용해라이언트 요청이 들어올 때부터 끝날 때 까지만 1차 캐시가 유효하다.

**1차 캐시 동작 방식**

- 최초 조회할 때는 1차 캐시에 엔티티가 없습니다.
- 데이터베이스에서 엔티티를 조회합니다.
- 엔티티를 1차 캐시에 보관합니다.
- 1차 캐시에 보관된 결과를 반환합니다.
- 이후 같은 엔티티를 조회하면 1차 캐시에 같은 엔티티가 있으므로 데이터베이스를 조회하지 않고 1차 캐시의 엔티티를 그대로 반환합니다.
- 1차 캐시는 객체의 동일성을 (a == b)를 보장합니다.

### JPA 2차 캐시

애플리케이션에서 공유하는 캐시 (JPA에서는 공유캐시, 일반적으로 2차 캐시라고 부름)

~~예) Redis, Memcached… (2차 캐시를 구현하는 기술 중 하나.)~~

2차 캐시는 애플리케이션 범위의 캐시이다. 따라서 애플리케이션을 종료할 떄까지 캐시가 유지된다. (분산 캐시, 클러스터링 환경의 캐시는 애플리케이션보다 더 오래 유지될 수 있다.)

![](/assets/img/2026-02-01/be-jpa-first-second-level-cache-3.png)

엔티티 매니저를 통해 데이터를 조회할 때 우선 2차 캐시에서 찾고, 없으면 DB에서 찾는다. 2차 캐시를 적절히 활용하면 DB 조회 횟수를 획기적으로 줄일 수 있다.

Hibernate를 포함한 대부분 JPA 구현체들은 **애플리케이션 범위의 캐시**를 지원하는데 이를 2차캐시라고 한다.

**2차 캐시 동작 방식**

- 영속성 컨텍스트는 엔티티가 필요하면 2차 캐시를 조회한다.
- 2차 캐시에 엔티티가 없으면 데이터베이스를 조회합니다.
- 결과를 2차 캐시에 보관합니다.
- 2차 캐시는 자신이 보관하고 있는 엔티티를 복사해서 반환합니다.
- 2차 캐시에 저장되어 있는 엔티티를 조회하면 복사본을 만들어 반환합니다.
- 2차 캐시는 데이터베이스 기본 키를 기준으로 캐시하지만 영속성 컨텍스트가 다르면 객체 동일성 (a == b)을 보장하지 않습니다.

![](/assets/img/2026-02-01/be-jpa-first-second-level-cache-4.png)

2차 캐시는 동시성을 극대화하기 위해 한 객체에 직접 반환하지 않고, 복사본을 만들어서 반환한다.

복사본을 반환하는 이유 : 캐시 객체를 그대로 반환하면 여러 곳에서 같은 객체를 동시에 수정하는 문제가 발생

→ 이를 해결하기 위해서 객체에 락을 걸면 성능 및 동시성이 떨어질 수 있다. 그래서 2차 캐시는 원본 대신 복사본을 반환한다. 2차 캐시를 잘 사용하면 1차 캐시의 동시성 문제를 해결하고 조회 성능을 끌어올릴 수 있다.

### 2차 캐시의 사용

```java
spring.jpa.properties.hibernate.cache.use_second_level_cache = true
// 2차 캐시 활성화합니다.

spring.jpa.properties.hibernate.cache.region.factory_class
// 2차 캐시를 처리할 클래스를 지정합니다.

spring.jpa.properties.hibernate.generate_statistics = true
// 하이버네이트가 여러 통계정보를 출력하게 해주는데 캐시 적용 여부를 확인할 수 있습니다.
```

```java
@Entity  // 2차캐시 활성화. true는 기본 값이라 생략가능 @Cacheable(value=true)
@Cacheable
public class Team {
	@Id @GeneratedValue
	private Long id;
	...
}
```

- cache mode

  ![](/assets/img/2026-02-01/be-jpa-first-second-level-cache-5.png)

@Cacheable의 속성 전략

- usage : CacheConcurrencyStrategy를 사용해서 캐시 동시성 전략을 설정합니다.
- region : 캐시 지역을 설정합니다.
- include : 연관 객체를 캐시에 포함할지 선택합니다. all, non-lazy 옵션을 선택할 수 있습니다. 기본값은 all 입니다.

캐싱 동시성 전략

- READ_ONLY : 자주 조회하고 수정 작업을 하지 않는 데이터에 적합합니다.
- READ_WRITE : 조회 및 수정 작업을 하는 데이터에 적합합니다. Phantom Read 가 발생할 수 있으므로 SERIALIZABLE 격리 수준에서는 사용할 수 없습니다.
- NONSTRICT_READ_WRITE : 거의 수정 작업을 하지 않는 데이터에 적합합니다.

**[멘토님 피드백]**

2차 캐시 - 여러 서버일 때 Sync 문제 발생

캐시 히트 비율이 다를 수 있다.

redis를 어떤 용도로 쓰는지

장애가 나면 롤백이 먼저

캐시의 가장 중요한 건 HitRate
