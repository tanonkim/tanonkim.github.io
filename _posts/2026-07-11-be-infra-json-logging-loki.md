---
title: "로그: JSON 표준화 + Grafana/Loki 수집"
date: 2026-07-11 10:55:00 +0900
categories: [infra, logging]
tags: [logging, loki, firelens]
---

> 실제 서비스명, 도메인, 리소스 식별자 등은 보안을 위해 예시 값으로 바꿨습니다.
{: .prompt-info }

### 무엇을 / 왜

로그를 JSON 한 줄/이벤트로 stdout에 찍고(구조화), 수집·저장·조회는 인프라가 담당한다.

- 왜 JSON: 텍스트 로그는 검색·필터가 어려움. JSON이면 필드(traceId·level·service)로 쿼리 가능.
- 앱은 벤더 중립: 앱은 stdout에 JSON만. 수집기(Datadog/CloudWatch/Loki)는 갈아끼울 수 있게.

### 결정 변경: Datadog → Grafana/Loki

체크리스트 원문은 "Datadog 로그 연동"이었으나, 수집을 Grafana/Loki(FireLens)로 변경. (앱의 JSON 로깅은 그대로, 목적지만 교체 — 벤더 중립의 실익)

앱 코드 — Spring Boot 4.1 내장 구조화 로깅 (추가 의존성 0)

**앱 코드**

```hcl
  # application.yml
  logging:
    structured:
      ecs:
        service:
          environment: ${APP_ENV:local}   # ECS/Grafana env 태그. service.name은 spring.application.name
  자동
    pattern:
      console: "%d{HH:mm:ss.SSS} [%X{traceId:-}] %-5level %logger{36} - %msg%n"  # 로컬용 텍스트
  # 운영에서만 JSON(ECS) 전환 (로컬은 텍스트 유지)
  LOGGING_STRUCTURED_FORMAT_CONSOLE=ecs
  APP_ENV=production
  - 로컬 = 사람이 읽는 텍스트, 운영 = JSON(ECS) — env 하나로 전환.
  - **traceId(MDC)**가 JSON 최상위 필드로 자동 노출 → 요청 추적 쉬움.
```

**수집: FireLens(Fluent Bit) → Loki (Terraform 토글)**

```hcl
  # ecs-service 모듈 — log_driver 토글
  # 기본 awslogs(CloudWatch, Loki 준비 전 배포 가능)
  container "my-api" {
    logConfiguration = var.log_driver == "firelens" ? {
      logDriver = "awsfirelens"
      options   = { Name = "loki", host = var.loki_host, port = "3100",
                    labels = "job=my-api-prod,env=production,service=my-api-prod" }
    } : {
      logDriver = "awslogs"   # CloudWatch
    }
  }
  # + firelens면 log_router(aws-for-fluent-bit) 사이드카 자동 추가
```
