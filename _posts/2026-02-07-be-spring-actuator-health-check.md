---
title: "Spring Boot Actuator의 헬스체크 살펴보기"
date: 2026-02-07 17:49:00 +0900
categories: [spring, core]
tags: [spring, actuator, health-check]
---

### 헬스체크

**로드밸런서**

서비스의 고가용성(HA, High Availability), 고성능을 위한 부하 분산 등의 이유로 우리는 서버의 이중화(혹은 그 이상)를 하고, 앞에서 어떤 서버로 요청을 보낼지 라우팅 역할을 하는 로드 밸런서를 둔다.

로드밸런서가 부하를 적절히 분산하여 A/B 서버 중 한 대에게 클라이언트의 요청을 보낸다.

가정 :

1. 만약 서버 B가 서비스 불가라면
2. 서버 증설 등 서버가 소스코드 로딩 상태라면

이 두 케이스 모두 해당 서버로 요청을 보내면 안된다. 정상적인 서비스가 불가하기에 클라이언트 요청을 수행할 수 없다.

⇒ 따라서 로드 밸런서에는 각 서버의 헬스체크 API를 호출해서 해당 서버가 현재 서비스가 가능한지 아닌지 주기적으로 점검한다.

헬스 체크 시 서버에 문제가 발견되면 로드밸런서는 해당 서버로 요청을 보내지 않게 된다.

결론 : **헬스 체크는 정상적으로 서비스가 가능한 서버에만 트래픽을 보내서 서비스의 고가용성을 확보**하는 데 도움이 된다.

### Spring Boot Actuator의 헬스체크

[Spring Boot Acutator](https://docs.spring.io/spring-boot/docs/3.0.5/reference/html/actuator.html)를 의존성으로 추가하면 기본적으로 헬스 체크 엔드포인트가 활성화된다.

Spring Boot 3.x 기준으로 헬스 체크 엔드포인트는 `/actuator/health`이고, 설정을 바꾸지 않아도 해당 엔드포인트로 접속하면 HTTP 200 상태 코드와 해당 서버의 상태가 Response Body로 응답된다.

**어떤 기준으로 서버의 헬스를 체크할까**

Spring Boot Actuator는 어떠 기준으로 서버의 헬스 체크를 할까?

확인하려면 [Health Information 문서](https://docs.spring.io/spring-boot/docs/3.0.5/reference/html/actuator.html#actuator.endpoints.health)를 살펴보면 된다(해당 정보는 보안에 민감한 요소가 들어있을 수 있어서 퍼블릭하게 접근이 가능해서는 안 된다)

![](/assets/img/2026-02-07/be-spring-actuator-health-check-1.png)

### 헬스체크에서 조심해야 하는 점

**의도하지 않은 장애 발생**

가정 : 서비스를 제공하는 서비스 DB, 데이터를 분석하는 로그 DB가 있다.

그리고 로그 DB에 적재하는 작업은 비동기로 별도의 스레드에서 처리하도록 작업을 해놨다 (로그 데이터 저장이 불가하더라도, 실시간 서비스에는 문제가 없어야 하기 때문)

로그 DB에 작업을 해야 해서 순단 or 접속에 문제가 생긴다면?

![](/assets/img/2026-02-07/be-spring-actuator-health-check-2.png)

1. `RoutingDataSourceHealthContributor`에 의해 여러 DataSource의 헬스를 체크를 한다.
   1. `DataSourceHealthIndicator`에 의해 “서비스 DB”의 상태를 체크했을 때는 `UP`이 반환
   2. `DataSourceHealthIndicator`에 의해 “로그 DB”의 상태를 체크했을 때는 `DOWN`이 반환
2. SimpleStatusAggregator에 의해 서비스 상태를 판단하는데, 순서를 설정하지 않았을 시 **DOWN이 하나라도 있으면 DOWN이 반환됨**
3. 서비스의 상태가 `DOWN`(`503`)으로 판단됐기 때문에 로드 밸런서에서는 서버로 트래픽을 보내지 않게 된다.
4. 서비스 DB에 문제가 없음에도 불구하고 클라이언트 요청은 처리되지 않고 장애가 발생한다.

**해결방법**

- Spring Boot Actuator의 헬스 체크가 아닌 직접 헬스 체크 API를 구현
- `HealthIndicator` 중에 헬스 체크에 영향을 끼치지 않길 희망하는 것들은 비활성화 (RDB를 예로 들자면 `management.health.db.enabled: false`(기본값 `true`)로 설정한다거나)
- 문제가 되는 `HealthIndicator` 빈을 직접 생성해서 Auto Configuration의 동작을 오버라이딩 하는 방법 등등

### TossTect 트러블슈팅의 지연 경험

예전에 API 서버에서 외부 의존성 중에 ES만 죽었는데, API 서버가 죽었다고 판단돼서 DOWN이 된적이 있었어요. 헬스 체크에서 detail 옵션을 키면, 상세하게 쭉 나오더라고요. 당시에 LB 통해서 접근이 안 됐는데, WAS는 개별로 접근했을 때는 문제가 없어 보여서 트러블 슈팅이 늦어졌었습니다.

⇒ ES(Elasticsearch) 서버가 죽었을 때 해당 서버의 헬스체크도 같이 죽게 된다는 걸 예측

(DOWN이 하나라도 있으면 DOWN)
