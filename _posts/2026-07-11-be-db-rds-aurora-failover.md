---
title: "RDS"
date: 2026-07-11 11:21:00 +0900
categories: [aws, rds]
tags: [aurora, hikaricp, failover]
---

> 실제 서비스명, 도메인, 리소스 식별자 등은 보안을 위해 예시 값으로 바꿨습니다.
{: .prompt-info }

RDS/Aurora **Cluster 모드 + Spring Boot**면 핵심은 이겁니다.

## 1. 절대 Instance Endpoint에 붙지 말기

장애조치 대응을 위해서는 반드시:

```
쓰기: cluster endpoint
읽기: reader endpoint
```

를 써야 합니다. AWS도 Aurora failover 시 **개별 DB instance endpoint가 아니라 cluster/reader endpoint 사용**을 권장합니다. ([Repost](https://repost.aws/knowledge-center/failovers-aurora-mysql))

```yaml
spring:
  datasource:
    url: jdbc:mysql://xxx.cluster-xxxx.ap-northeast-2.rds.amazonaws.com:3306/app
```

개별 인스턴스 엔드포인트:

```
xxx-instance-1.xxxx.ap-northeast-2.rds.amazonaws.com
```

이런 식으로 붙으면 failover 후에도 죽은 writer를 바라볼 수 있습니다.

---

## 2. 일반 JDBC만 쓰면 DNS 전환 지연이 생길 수 있음

Aurora failover는 endpoint DNS가 바뀌는 구조입니다. 문제는 JVM, OS, 커넥션풀, 드라이버가 DNS를 오래 캐싱하면 전환이 늦어집니다. Aurora endpoint TTL은 5초지만 클라이언트 환경 때문에 지연될 수 있습니다. ([Amazon Web Services, Inc.](https://www.amazonaws.cn/en/blog-selection/improve-application-availability-on-amazon-aurora/))

JVM DNS TTL을 낮게 잡는 게 좋습니다.

```bash
-Dnetworkaddress.cache.ttl=5
-Dnetworkaddress.cache.negative.ttl=0
```

또는 Java security 설정:

```
networkaddress.cache.ttl=5
networkaddress.cache.negative.ttl=0
```

---

## 3. HikariCP는 “기존 죽은 커넥션”을 빨리 버리게 해야 함

Failover 순간에 이미 풀에 들어있는 커넥션은 새 writer로 자동 변신하지 않습니다. 그래서 Hikari가 오래 물고 있으면 장애가 길어집니다.

추천값 예시:

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 30
      minimum-idle: 5
      connection-timeout: 3000
      validation-timeout: 1000
      idle-timeout: 600000
      max-lifetime: 1740000
      keepalive-time: 300000
```

주의점:

| 옵션 | 권장 |
|---|---|
| `connection-timeout` | 2~5초 |
| `validation-timeout` | 1초 |
| `max-lifetime` | DB wait_timeout보다 짧게 |
| `minimum-idle` | 너무 크게 잡지 않기 |
| `maximum-pool-size` | ECS task 수 × pool size 기준으로 계산 |

`max-lifetime`을 너무 길게 두면 장애 전 커넥션이 오래 살아남습니다.

---

## 4. 가장 좋은 방식은 AWS Advanced JDBC Wrapper 사용

Aurora MySQL/PostgreSQL이면 **AWS Advanced JDBC Wrapper Driver**를 쓰는 게 제일 깔끔합니다. 이 드라이버는 **Aurora topology를 캐싱하고 writer/reader 역할을 직접 감지해서 DNS 지연을 우회합니다.** AWS도 Aurora의 failover 기능을 제대로 활용하려면 AWS JDBC Driver 사용을 권장합니다. ([Amazon Web Services, Inc.](https://aws.amazon.com/blogs/database/unlock-amazon-auroras-advanced-features-with-standard-jdbc-driver-using-aws-advanced-jdbc-wrapper/))

MySQL 예시:

```yaml
spring:
  datasource:
    driver-class-name: software.amazon.jdbc.Driver
    url: jdbc:aws-wrapper:mysql://xxx.cluster-xxxx.ap-northeast-2.rds.amazonaws.com:3306/app
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    hikari:
      connection-timeout: 3000
      validation-timeout: 1000
      max-lifetime: 1740000
```

Gradle 예시:

```
implementation("software.amazon.jdbc:aws-advanced-jdbc-wrapper:2.5.6")
implementation("com.mysql:mysql-connector-j:8.4.0")
```

PostgreSQL이면:

```
implementation("software.amazon.jdbc:aws-advanced-jdbc-wrapper:2.5.6")
implementation("org.postgresql:postgresql:42.7.4")
```

---

## 5. RDS Proxy도 선택지

애플리케이션 수정 최소화가 목적이면 **RDS Proxy**도 좋습니다. AWS 문서 기준 RDS Proxy는 failover 시 standby DB instance로 자동 연결하고, 애플리케이션 커넥션 보존에 도움을 줍니다. ([AWS Documentation](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/rds-proxy.html))

다만 주의점이 있습니다.

| 방식 | 장점 | 주의점 |
|---|---|---|
| Cluster Endpoint + Hikari | 단순 | DNS/커넥션풀 영향 큼 |
| AWS Advanced JDBC Wrapper | failover 빠름 | 드라이버 설정 검증 필요 |
| RDS Proxy | 앱 수정 적음, 커넥션 보호 | Proxy 비용, 세션 pinning 주의 |

---

## 6. Read/Write 분리 시 가장 조심할 점

Spring에서 `@Transactional(readOnly = true)` 기준으로 reader endpoint를 쓰게 만들 수 있는데, 아래는 조심해야 합니다.

- a라는 테이블 write 를 하고 read를 한다음에 연산을해서 b라는 테이블에 write 를 할경우
  - rag 이슈

### 문제 상황

```kotlin
@Transactional
fun createOrder() {
    orderRepository.save(order)

    val result = orderRepository.findById(order.id)
}
```

쓰기 직후 읽기를 reader endpoint로 보내면 replica lag 때문에 방금 쓴 데이터가 안 보일 수 있습니다.

그래서 원칙은:

| 상황 | 연결 |
|---|---|
| 쓰기 트랜잭션 내부 읽기 | writer |
| 로그인/결제/청구 상태 확인 | writer 권장 |
| 단순 목록/통계 조회 | reader 가능 |
| read-after-write 필요한 API | writer |

---

## 7. 장애조치 테스트는 꼭 해야 함

운영 반영 전에 반드시 RDS에서 failover를 강제로 발생시켜야 합니다.

```bash
aws rds failover-db-cluster \
  --db-cluster-identifier your-cluster-id
```

확인할 것:

- API 장애 시간이 몇 초인지
- Hikari pool이 정상 회복되는지
- ECS task 재시작 없이 복구되는지
- `Communications link failure`, `read-only transaction`, `connection is closed` 로그 패턴
- 쓰기 API가 read replica로 붙는 문제 없는지

---

## 추천 결론

운영 안정성 기준이면:

```
1순위: AWS Advanced JDBC Wrapper + cluster endpoint
2순위: RDS Proxy + HikariCP 튜닝
3순위: 일반 JDBC + DNS TTL + HikariCP 튜닝
```

Spring Boot에서 최소 설정으로 가려면 **RDS Proxy**가 편하고, failover 시간을 진짜 줄이고 싶으면 **AWS Advanced JDBC Wrapper**가 더 적합합니다.
