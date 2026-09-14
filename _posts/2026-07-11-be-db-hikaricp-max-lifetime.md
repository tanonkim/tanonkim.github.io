---
title: "HikariCP max-lifetime (커넥션 풀 수명 관리)"
date: 2026-07-11 10:42:00 +0900
categories: [database, connection-pool]
tags: [hikaricp, mysql]
---

> 실제 서비스명, 도메인, 리소스 식별자 등은 보안을 위해 예시 값으로 바꿨습니다.
{: .prompt-info }

### 무엇을 / 왜

HikariCP는 DB 커넥션 풀. 핵심 목적은 이미 끊긴 커넥션을 앱이 재 사용해서 Communications link failure를 예방하는 것

왜 끊기는가 : DB MySQL wait_timeout, 중간경로(NAT/방화벽/ELB)가 유휴 커넥션을 조용히 끊음 → 쿼리 요청 시 터짐 → **앱이 DB 커넥션보다 먼저 커넥션을 폐기하도록 수명 맞추기**

### 설정(application.yml)

```yaml
  spring:
    datasource:
      hikari:
        pool-name: my-api-pool
        maximum-pool-size: ${DB_POOL_MAX_SIZE:10}            # 상한
        minimum-idle:      ${DB_POOL_MIN_IDLE:10}            # min=max → 고정 풀
        connection-timeout: ${DB_CONNECTION_TIMEOUT_MS:3000}        # 3s: 풀 고갈 시 빠른 실패
        validation-timeout: ${DB_VALIDATION_TIMEOUT_MS:2000}        # 2s: 커넥션 검증 (<
  connection-timeout)
        keepalive-time:     ${DB_KEEPALIVE_MS:120000}               # 2분: 유휴 커넥션 주기적 깨움
        max-lifetime:       ${DB_MAX_LIFETIME_MS:1740000}           # 29분: 커넥션 최대 수명
```

### 핵심 순서

- HikariCP max-lifetime(29분) < MySQL wait_timeout(RDS 파라미터 그룹 : 60분)
  - 앱이 커넥션을 29분에 스스로 폐기 → DB가 60분에 끊기 전에 미리 새로 교체.
  - 그래서 RDS 파라미터그룹에서 MySQL wait_timeout=3600을 max-lifetime보다 크게 잡음
  - 경로상 유휴 컷(NAT/방화벽/ELB)보다도 짧아야
- HikariCP keepalive-time(2분) < HikariCP max-lifetime
  - 2분마다 유휴 커넥션에 가벼운 핑 - NAT/방화벽이 유휴로 판단해 끊는걸 예방
- validation-timeout(2s) < connection-timeout(3s)
  - 풀이 고갈되었을 때 요청이 무한대기 하지 않고, 3초만에 실패 → 장애가 앱 전체로 번지는 것 방지(`fail-fast`)

### pool-size 산정 (ECS 태스크 수 기준)

전체 DB 커넥션 = (ECS 태스크 수) × (maximum-pool-size) ≤ RDS max_connections

예) 태스크 6개(오토스케일 max) × 10 = 60 커넥션 ≤ RDS max_connections

그래서 오토스케일 max와 pool-size를 같이 봐야 RDS 커넥션 한도를 안 넘긴다. db.t4g.small의 max_connections(기본 ~수백)와 비교해 여유 확인. pool-size는 변수 (DB_POOL_MAX_SIZE)라 태스크 수 확정 후 조정.
