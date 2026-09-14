---
title: "Redis"
date: 2026-07-11 11:02:00 +0900
categories: [infra, redis]
tags: [redis, cache, replication]
---

> 실제 서비스명, 도메인, 리소스 식별자 등은 보안을 위해 예시 값으로 바꿨습니다.
{: .prompt-info }

### 무엇을 / 왜

Redis = 캐시(Cache-Aside). 자주 읽는 데이터를 캐시에 두어 DB 부하·응답시간↓.

- 예: admin KPI 대시보드 등 무거운 집계를 짧은 TTL로 캐시.
- 온프렘 선택: on-prem A = master, on-prem B = replica. (AWS ElastiCache 안 씀 — 비용)

### 구성

```
      AWS ECS (앱)
          │  REDIS_HOST → on-prem A master (VPN)
          ▼
 on-prem A Redis (master)  ──replication──▶  on-prem B Redis (replica, HA)
```

앱은 master 단일 접속(Cache-Aside 읽기/쓰기).

```hcl
  # application.yml
  spring:
    data:
      redis:
        host: ${REDIS_HOST:localhost}   # 운영: on-prem A master
        port: ${REDIS_PORT:6379}
```

### 온프렘 구축 요건

| 항목 | 내용 |
|---|---|
| 복제 | on-prem B 노드에 replicaof `<master-host>` 6379. 확인 INFO replication |
| 네트워크 | ECS 서브넷, on-prem B 노드 → on-prem A master:6379 도달 = VPN/Tunnel(Tailscale) |
| 보안 | requirepass + TLS. 방화벽으로 ECS SG, Onpremise만 |
| 자동/장애조치 | master 죽을 때 replica 자동 승격 = Redis Sentinel |

### Latency 고려사항

캐시는 앱과 네트워크상 가까워야 빠르다. AWS 앱 ↔ on-prem A Redis는 WAN을 건넌다 → 왕복 수십 ms 가능.

- 이러면 캐시 읽기가 AWS 내 RDS 쿼리보다 느려져 캐시의 의미가 줄 수 있음.
- 검증 필요: 실제 RTT를 재보고, 캐시 이득 > WAN 지연인지 확인. 지연이 크면 (a) 지연에 둔감한 데이터만 캐시, (b) 필요 시 AWS ElastiCache 재검토.
