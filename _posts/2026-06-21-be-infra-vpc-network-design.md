---
title: "VPC / 네트워크 아키텍처 설계 (서비스 B)"
date: 2026-06-21 09:32:00 +0900
categories: [aws, vpc]
tags: [aws, vpc, network]
---

> 실제 서비스명, 도메인, 리소스 식별자 등은 보안을 위해 예시 값으로 바꿨습니다.
{: .prompt-info }

## 서비스 B

사업 설명 필요

### 1. 개요/목표/클라우드 경계

구성 4요소

| **구성** | **핵심 역할** |
|---|---|
| **① 쇼핑몰** | 판매자 · 관리자 · 구매자 3개 역할의 주문/상품/정산 |
| **② 프라이싱 매니저** | 한 상품의 자사몰/11번가/쿠팡 **최저가 비교** → 자사몰 구매 유도 |
| **③ 영상 상품분석** | 영상 속 상품 인식·추출 **온프레미스 GPU** |
| **④ 데이터** | 메인 DB(MySQL) **온프레미스** · Redis 필수 |

클라우드 ↔ Onpremise

| **레이어** | **위치** |
|---|---|
| 웹/앱 · ALB · ASG | **AWS** 프라이빗-앱 |
| 이벤트 백본(Kafka) | **AWS** 프라이빗-데이터 |
| 선착순 핫패스 Redis | **AWS** 권장 (§3 결정) |
| 메인 DB(MySQL)·영상분석 | **온프레미스** · VPN 연동 |

### 2. 전체 시스템 아키텍쳐

> 보안상 이미지를 생략했습니다.

| **서비스 B 구성요소** | **AWS / 온프렘 매핑** | **티어** |
|---|---|---|
| 구매자 프론트엔드 (웹/앱) | CloudFront + S3 → External ALB → EC2 | Public/App |
| 판매자/관리자 포털 | Internal ALB → EC2(admin), 관리자 망 제한 | App |
| 앱 서버(주문,상품,쿠폰 등) | EC2 ASG(Spot인스턴스), 공인IP X | App |
| 캐시, 세션, 선착순 카운터(쿠폰 등) | Redis | Data |
| 이벤트 백본 | Kafka (AWS MSK Provisioned, kafka.t3.small) | Data |
| 메인 DB | 예상) Onpremise MySql, Master/Slave, VPN | On-Premise |
| 프라이싱 크롤러 | EC2/ECS 워커 → NAT egress(고정 EIP) | App |
| 프라이싱 위젯 | S3(my-bucket) + CloudFront | Edge |
| 영상 상품 분석 | Onpremise GPU | On-Premise |
| 정적자산 업로드 | S3 + CloundFront + VPC Gateway Endpoint로 NAT 우회 | Data |
| 접근, 운영 | SSM, IAM | 전 Tier |

앱서버에서 공인IP가 없는 이유

EC2/ECS 워커 → NAT egress(고정 EIP) → 이거 용어정리

S3 + CloundFront + VPC Gateway Endpoint로 NAT 우회 → 이해 잘 못함

3-Tier VPC) 블록 10.xx.xx.xx/20 / 블록 10.xx.xx.xx/20 → 차이점??

이거 3tier가 맞는지? (위에 이미지)

### 3. 3-Tier VPC / 네트워크 상세설계

> 보안상 이미지를 생략했습니다.

**3.1 신규 VPC 별도 생성**

현재 서비스 C 10.xx.xx.xx/16, 기본 VPC 172.31.0.0/16과 겹치지 않게 VPC 분리 생성

신규 VPC : 10.xx.xx.xx/16

| **RFC1918 사설 대역** | **클래스** | **크기** | **서비스 B에서** |
|---|---|---|---|
| `10.0.0.0/8` | A | 가장 큼 (/16을 256개 배정 가능) | **채택** — 여러 VPC·피어링·온프렘을 비충돌로 확장 |
| `172.16.0.0/12` | B | 중간 | 기본 VPC가 172.31 사용 중 → 회피 |
| `192.168.0.0/16` | C | 가장 작음 | 온프렘 사설망이 주로 사용 → 회피 |

**C클래스 주로 사용할텐데 다시 확인**

**무슨 클래스 사용하지 않는지 다시 확인**

/16이 C클래스?

**3.2 서브네팅 - CIDR로 가용 IP 산출**

| **CIDR** | **총 주소 = 2의 거듭제곱** | **AWS 가용 IP** | **쓰임** |
|---|---|---|---|
| `/16` | 2¹⁶ = 65,536 | ~65,531 | VPC 전체 대역 |
| `/20` | 2¹² = 4,096 | 4,091 | AZ별 블록(여유) |
| `/24` (C클래스) | 2⁸ = 256 | **251** | 티어별 서브넷 |
| `/28` | 2⁴ = 16 | 11 | 최소 단위(엔드포인트 등) |

- C클래스 251개 :
  - .0 : 네트워크 주소
  - .1 : VPC 라우터
  - .2 : Amazon DNS
  - .3 : 향후 예약
  - .255 : 브로드캐스트

**3.3 서브넷 배정표**

| **티어** | **AZ 2a** | **AZ 2c** | **가용 IP** | **들어갈 것** |
|---|---|---|---|---|
| **퍼블릭** | `10.xx.xx.xx/24` | `10.xx.xx.xx/24` | 각 251 | External ALB, NAT GW(+EIP) |
| **프라이빗-앱** | `10.xx.xx.xx/24` | `10.xx.xx.xx/24` | 각 251 | EC2 ASG(main·admin), Internal ALB, 크롤러, Redis(핫패스) |
| **프라이빗-데이터** | `10.xx.xx.xx/24` | `10.xx.xx.xx/24` | 각 251 | MSK 브로커, VPN GW/Router, 상태저장 노드 |

- public : 퍼블릭 인터넷에 직접 닿아야 하는 것만

  External ALB, NAT Gateway(+EIP), (필요 시 Bastion). 앱서버 X

- private-data : 완전 격리

  DB, 상태 저장은 앱 티어에서만 저장, 인터넷 경로 없음

  서비스 B는 메인 DB가 Onpremise → 해당 티어가 Onpreimse DB로 가는 사설 게이트웨이 계층

  MSK 브로커 위치

- priavate-app

  EC2는 공인 IP없이 배치, 밖으로 나갈 땐 NAT 경유

  ALB를 통해서만 인입

**3.4 IGW · 라우팅 테이블**

- IGW는 VPC당 1개
- Public RT가 0.0.0.0/0 → IGW를 가질 때 양방향 인터넷이 열린다.

| **라우팅 테이블** | **대상(destination) → 타깃(target)** | **의미** |
|---|---|---|
| **퍼블릭 RT** | `10.xx.xx.xx/16` → local<br>`0.0.0.0/0` → **IGW** | 인터넷 양방향. ALB·NAT만 거주 |
| **앱 RT (2a)** | local<br>`0.0.0.0/0` → **NAT-a**<br>온프렘 CIDR → VPN<br>`10.xx.xx.xx/16` → pcx(피어링)<br>`s3 prefix` → S3 Gateway EP | 인입 불가 / egress는 NAT / DB는 VPN / 기존 VPC는 피어링 / S3는 무료 EP |
| **앱 RT (2c)** | local · `0.0.0.0/0` → **NAT-c** · 온프렘 → VPN | AZ별 NAT로 포트고갈·장애 격리 |
| **데이터 RT** | local · 온프렘 CIDR → VPN **(인터넷 경로 없음)** | 완전 격리 — 밖으로 못 나가고 앱에서만 닿음 |

**3.5 접근제어 - SG 체인 + NACL 이중화**

SG(Stateful, 인스턴스 단위는 출처를 CIDR이 아닌, 상위 SG로 등록해 IP가 바뀌어도 자동 추적

NACL은 광역 차단으로 한겹 더 덮는다.

| **계층** | **Security Group (stateful) — 인바운드** | **NACL (stateless)** |
|---|---|---|
| 퍼블릭/ALB | alb-sg : 443(80 리다이렉트) ← 0.0.0.0/0 | 80/433 in, ephemeral out 허용 |
| 앱 | app-sg : 앱포트(예) 8000, 3000) ← alb-sg / internal-alb-sg | 퍼블릿 서브넷 CIDR + ephemeral(1024-65535) |
| 데이터 | data-sg: MySQL 3306/관리포트 ← app-sg에서만<br>msk-sg: 9092/9094 ← app-sg<br>redis-sg: 6379 ← app-sg | 앱 서브넷 CIDR만 허용, 그 외 deny |

**3.7 NAT Gateway 배치 + 크롤링/외부 호출 egress**

NAT은 프라이빗이 밖으로 나갈때만 쓰는 단방향 출구이다.

1. 인입(ALB → 앱 응답) : 세션이 유지돼 NAT이 불필요함
2. 앱이 먼저 밖으로 나갈 때 : NAT이 필요, 예) 외부 API, 크롤링, PG 등등

| **흐름** | **경로** | **NAT?** |
|---|---|---|
| 구매자 → 서비스 | CloudFront → ALB → 앱 | X |
| 크롤러 → 11번가/쿠팡 | 크롤러앱 → NAT → IGW → 외부몰 | O |
| 앱 → PG/외부 API | 앱 → NAT → IGW | O |
| 앱 → S3 | S3 Gateway Endpoint (NAT 우회) | X |

### 이벤트 드리븐 설계

"Kafka가 지금 안 들어가더라도 **바로 붙을 수 있게** 이벤트 드리븐으로."

동기 핫패스는 Redis, 비동기는 Kafka

그 사이를 Outbox로 느슨하게 연결

**Redis (필수 · 동기 핫패스)**

선착순 수량을 **원자적으로** 차감해 초과 발급을 원천 차단. `DECR` 또는 Lua 스크립트로 "확인+차감"을 한 번에 처리(레이스 컨디션 없음).

**Kafka (이벤트 백본 · 비동기)**

발급 성공 사실을 **이벤트로 흘려** 여러 컨슈머가 독립적으로 처리. 토픽 예시:

| **토픽** | **소비자** |
|---|---|
| `coupon.issued` | 쿠폰 영속화 → 온프렘 DB |
| `order.created` | 주문·결제·재고 |
| `price.updated` | 위젯 데이터 갱신 |
| `video.analyzed` | 카탈로그 등록 |

![](/assets/img/2026-06-21/be-infra-vpc-network-design-1.png)
