---
title: "DB 연결과 Failover"
date: 2026-08-10 19:12:00 +0900
categories: [database, connection-pool]
tags: [hikaricp, aurora, failover]
---

> 실제 서비스명, 도메인, 리소스 식별자 등은 보안을 위해 예시 값으로 바꿨습니다.
{: .prompt-info }

### AWS Advanced JDBC Wrapper의 역할

AWS Advanced JDBC Wrapper : DNS가 새 주소를 알려줌 + Pool이 죽은 새 커넥션을 걷어내기까지 ⇒ 이 두 시간을 줄이는 것(토폴로자 캐싱)

커넥션 풀이 붙잡고 있는건 **엔드 포인트**가 아닌, **이미 열린 TCP** 연결

failover이 발생하면 그 연결들이 한번에 죽고, 새로 붙을 때까지 DNS를 조회함. → 순단(중지된 현상)의 길이는 DNS가 새 주소 알려줌 + 풀이 죽은 커넥션을 걷어내기까지.

| 항목 | 기본값 | 브랜치 값 | 기본값의 문제 |
|---|---|---|---|
| `connection-timeout` | 30,000ms | 3,000ms | DB가 느려지거나 failover 중일 때 요청 스레드가 30초씩 풀 대기에 물립니다. 가상 스레드라 스레드는 안 마르지만, ALB idle(60s)·클라이언트 타임아웃 안에서 요청이 줄줄이 쌓여 장애가 증폭됩니다. **빨리 실패해야 빨리 회복합니다.** |
| `keepalive-time` | **0 (비활성)** | 120,000ms | 기본은 keepalive를 아예 안 합니다. NAT 인스턴스(fck-nat)·방화벽이 유휴 커넥션을 조용히 끊으면, 풀은 그걸 모른 채 죽은 커넥션을 빌려주고 `Communications link failure`가 간헐적으로 터집니다. 재현이 어려워 디버깅이 제일 괴로운 부류입니다. |
| `max-lifetime` | 1,800,000ms (30분) | 1,740,000ms (29분) | 기본값도 MySQL `wait_timeout`(28800s)보다는 짧아서 그 자체로 위험하진 않습니다. 다만 terraform이 `wait_timeout`을 3600s로 낮추므로 **명시해두는 게 맞습니다.** |
| `minimum-idle` | = max (이미 고정 풀) | 10 | 기본도 이미 min=max입니다. 명시가 문서 역할. |
