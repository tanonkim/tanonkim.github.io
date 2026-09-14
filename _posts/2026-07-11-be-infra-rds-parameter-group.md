---
title: "Parameter Group"
date: 2026-07-11 10:49:00 +0900
categories: [aws, rds]
tags: [rds, mysql, parameter-group]
---

> 실제 서비스명, 도메인, 리소스 식별자 등은 보안을 위해 예시 값으로 바꿨습니다.
{: .prompt-info }

## 파라미터 그룹 초안 튜닝 (max_connections / wait_timeout / lock wait)

### 무엇을 / 왜

RDS 파라미터 그룹 = DB 엔진(MySQL) 설정값 묶음. 기준은 `웹 응답 2초 내` — 요청이 커넥션·락에서 오래 멈추지 않게 한다.

### Terraform 설정

```hcl
  resource "aws_db_parameter_group" "this" {
    family = "mysql8.0"
    parameter { name = "wait_timeout"        value = "3600" }   # ✅ 앱 max-lifetime(29분)보다 크게(1h)
    parameter { name = "interactive_timeout" value = "3600" }
  }
```

**wait_timeout / interactive_timeout = 3600s ✅**

- 유휴 커넥션을 DB가 끊는 시간. 앱 HikariCP max-lifetime(29분)보다 크게 둬서, 앱이 먼저 커넥션을 폐기하도록(끊긴 커넥션 재사용 방지 — HikariCP 항목과 연결).

**max_connections — 기본값 사용 (산정만)**

- `db.t4g.small`은 메모리 기반 공식으로 자동 산정(대략 수백). 명시 오버라이드 안 함(기본으로 충분).
- 산정 기준: 전체 커넥션 = (ECS 태스크 수) × (HikariCP pool 10) ≤ max_connections.

  예) 태스크 6 × 10 = 60 → 여유. 태스크·pool 확정 후 초과하면 그때 상향.

**lock wait — 결정 포인트 (아직 미조정)**

- innodb_lock_wait_timeout 기본 50초. `2초 내 응답` 기준이면 락 대기가 50초씩 매달리는 게 문제.
- 트레이드오프:
  - 낮추면(예: 몇 초) → 락 경합 시 빨리 실패(요청이 안 매달림, 재시도 유도).
  - 너무 낮추면 → 정상적인 짧은 경합에도 불필요한 timeout 에러.
- 현재는 기본 유지. "웹 응답 2초 내"를 엄격히 하려면 커넥션(Hikari 3s)·쿼리·락 타임아웃을 함께 낮추는 초안이 필요 → 부하 보고 조정.

"웹 응답 2초 내"를 위한 타임아웃 정렬 (큰 그림)

요청 → [커넥션 획득: Hikari connection-timeout 3s]<br>
→ [락 대기: innodb_lock_wait_timeout (기본 50s → 낮출지 결정)]<br>
→ [쿼리 실행: 필요시 statement timeout]

- 이미 있음: Hikari 3s fail-fast, ALB/tomcat 타임아웃 정합(별도 항목).
- 조정 여지: 락 대기·쿼리 타임아웃.
