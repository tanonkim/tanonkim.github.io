---
title: "RDS (서비스 A 적용기)"
date: 2026-07-11 10:23:00 +0900
categories: [aws, rds]
tags: [rds, mysql, terraform]
---

> 실제 서비스명, 도메인, 리소스 식별자 등은 보안을 위해 예시 값으로 바꿨습니다.
{: .prompt-info }

## RDS 연결 방식 (single-AZ + 순수 드라이버 + 온프렘 replica)

### 결정 이유

[처음 전제]  Aurora 클러스터(writer+reader) + failover2<br>
↓ (실제 구성 확인)<br>
[실제]  master = 일반 RDS MySQL, replica = 온프렘 MySQL (읽기 분산)<br>
↓ (비용)<br>
[결정]  single-AZ RDS 인스턴스 + 순수 jdbc:mysql 드라이버

**왜 JDBC Wrapper failover2를 안 쓰나**

- AWS Advanced JDBC Wrapper의 failover2·readWriteSplitting은 **Aurora/RDS** 클러스터 토폴로지 API로 노드를 발견한다.
- 우리 replica는 온프렘 MySQL → 그 토폴로지에 안 잡힌다. 래퍼가 온프렘을 볼 수 없음.
- 게다가 **single-AZ 인스턴스엔 장애조치로 넘길 노드가 없어** 래퍼 이득이 사실상 0.
- ∴ 런치는 순수 jdbc:mysql://. 래퍼 의존성은 유지해서 나중에 Multi-AZ로 올리면 env만 교체(jdbc:aws-wrapper:...?wrapperPlugins=efm2)로 켤 수 있게.

**토폴로지 & 읽기 분산**

```
           Application
                │
     AbstractRoutingDataSource   ← 앱측 라우팅 (온프렘 replica는 래퍼로 못 함)
      ┌─────────┴─────────┐
 Write DataSource    Read DataSource
      │                   │
 RDS MySQL(Master)    On-Prem MySQL(Replica)
      │
      └── binlog replication ──▶ On-Prem
```

- 온프렘 replica 읽기 분산 = **래퍼가 아니라 `앱`측** AbstractRoutingDataSource (writer=RDS, reader=온프렘).<br>@Transactional(readOnly)로 라우팅. ⚠️ read-after-write는 writer(복제 지연 stale 방지).
- **현재는 보류** — 읽기 부하가 실제 병목이 될 때 도입(env-gated: SERVICE_A_DB_READER_URL 없으면 단일 폴백).

### 장애대응

| 장애 | 대응 |
|---|---|
| Writer(RDS) 장애 | Multi-AZ 인스턴스면 AWS가 자동 승격, 앱은 재연결만<br>Single-AZ면 자동 failiover 없음ㄴ |
| 호스트 장애 | RDS Auto-Recovery가 자동 복구(대신 수분이 소요됨) |
| AZ 전체 장애 | single-AZ는 못넘김 → 백업/PITR 복원(다른 AZ로) |
| Onpremise Replica를 Writer로 승격 | 하면 안 됨. RDS가 source of truth인데 온프렘을 writer로 올리면 역전, <br>쓰기 failover 대상이 아님 |

### single-AZ 선택 이유:

상시 Multi-AZ는 스탠바이가 24시간 떠서 비용 2배.

저트래픽 초기엔 비용 우선 + 백업기반 복구(AZ장애 시 복원시간만큼 다운 감수)가 합리적. <br>RTO 안 되면 그때 Multi-AZ(env만 교체).

### Terraform RDS 모듈

```java
  resource "aws_db_instance" "this" {
    engine         = "mysql"
    instance_class = "db.t4g.small"      # 저비용 ARM
    multi_az       = false               # single-AZ (토글)
    storage_encrypted = true

    # 마스터 암호를 RDS가 Secrets Manager로 생성·로테이트 → TF state에 password 없음!
    manage_master_user_password = true

    vpc_security_group_ids = [var.rds_security_group_id]  # 3306 from app-sg만
    db_subnet_group_name   = aws_db_subnet_group.this.name # db 티어(격리 서브넷)

    backup_retention_period = 14         # 자동백업/PITR 14일
    deletion_protection     = true
    skip_final_snapshot     = false
  }

  resource "aws_db_parameter_group" "this" {
    family = "mysql8.0"
    parameter { name = "wait_timeout"        value = "3600" }  # ← 앱 max-lifetime(1740s)보다 크게
    parameter { name = "interactive_timeout" value = "3600" }
  }
```

### 개념들

| 개념 | 설명 |
|---|---|
| manage_master_user_password | RDS가 마스터 암호를 Secrets Manager에 직접 생성, 자동 로테이트. TF State에 비밀번호가 남지 않음 + 앱은 별도 최소권한 계정 사용 |
| wait_timeout > max-lifetime | DB 커넥션을 끊는 시간(wait_timeout)보다 <br>앱이 커넥션을 폐시하는 시간(HikariCP max-lifetime 29분)이 먼저 와야 끊긴 커넥션 재사용 에러 방지 |
| db_subnet_group 격리 | RDS는 인터넷 경로 없는 DB 티어 서브넷만. 외부 직접 접근 불가 |
| PITR (Point in Time Recovery) | 자동 백업 + 트랜잭션 로그로 임의 시점 복원 (오배포 손상 시점 직전으로) |
| efm2(host monitoring) | Multi-AZ 승격 시 끊긴 커넥션을 빨리 감지, 차단해 hang없이 재연결, single-AZ엔 불필요 |

멘토 체크리스트 대응

- RDS 연결 방식 택1: JDBC Wrapper(의존성 유지) vs Proxy → Wrapper 채택, 단 런치는 순수 드라이버
- RDS 종류 확정: 일반 RDS MySQL 단일 인스턴스(Aurora 아님)
- 읽기 라우팅: 온프렘 replica → 앱측 AbstractRoutingDataSource(보류)
- RDS 백업: 자동백업 14일·PITR·deletion protection
