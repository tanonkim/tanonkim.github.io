---
title: "3-Tier VPC / 네트워크 아키텍처 설계"
date: 2026-06-26 18:06:00 +0900
categories: [aws, vpc]
tags: [aws, vpc, network]
---

> 실제 서비스명, 도메인, 리소스 식별자 등은 보안을 위해 예시 값으로 바꿨습니다.
{: .prompt-info }

## 서비스 A

### 1. 요약

3-tier를 기술 스택에 그대로 매핑한다 — **웹(Presentation) = Next.js**, **앱(Application) = Spring Boot**, **데이터(Data) = MySQL**.

모든 컴퓨팅·DB는 **프라이빗 서브넷**, 인터넷 인입은 **CloudFront·ALB**로만, 외부 호출은 **NAT**로만.

VPC 대역 : 10.xx.xx.xx/16<br>티어 서브넷 : /24 · 251 IP<br>DB : MySQL :3306<br>DB 구성 :

- A안 RDS

> 보안상 이미지를 생략했습니다.

**~~B안 온프레미스 제외~~**

RDS 대신 **VPN 게이트웨이(VGW)**를 두고, 실제 MySQL master/slave는 온프렘에 둔 뒤 Site-to-Site VPN or Tailscale Subnet Router으로 잇는다.

### **2. 신규 VPC 생성 & 대역(CIDR) 선정**

- 신규 VPC 생성
  - 기존 서비스 C VPC(10.xx.xx.xx/16)사용 X
  - VPC 독립 생성
    - 서비스 생애주기, 과금, 장애, IAM 권한 분리
    - 만약 공유를 하면 한쪽 장애가 다른 서비스로 전파
    - 나중에 필요한 만큼만 VPC Peering 연결
- 10.x 사설대역
  - VPC는 인터넷에 직접 라우팅되지 않는 사설(RFC 1918) 대역 사용
  - 선택기준 : 미래에 다른 네트워크와 IP가 겹치지 않을 것
    - IP가 겹친다면 라우터가 어디로 갈지 X → 피어링, VPN 자체가 불가능해짐

| **사설 대역** | **크기** | **특징 / 충돌 위험** | **채택** |
|---|---|---|---|
| `10.0.0.0/8` | 약 1,677만 개 | 가장 넓다 → 옥텟을 서비스별로 계획 배정하기 좋음 | **채택** |
| `172.16.0.0/12` | 약 104만 개 | **AWS 기본 VPC가 172.31.0.0/16** 사용 → 충돌 위험 | 회피 |
| `192.168.0.0/16` | 65,536 개 | 공유기·사무실·Tailscale 등에서 흔히 사용 → 충돌 위험 | 회피 |

- 대역 할당 레지스트리 for 충돌방지

| **대역** | **용도** | **비고** |
|---|---|---|
| `10.xx.xx.xx/16` | 서비스 C 현행 VPC | 기존 운영 |
| `10.xx.xx.xx/16` | 서비스 B 신규(TO-BE) VPC | 기존 설계서 예약 |
| `10.xx.xx.xx/16` | **본 프로젝트 VPC** | 위 둘과 비충돌 → 피어링 가능 |

### 3. **서브네팅 설계 & CIDR 계산**

- 가용 IP 계산

  여기서 AWS는 서브넷마다 5개를 예약하므로 뺀다.

  /24 가용 IP : 2^8 - 5 = 251

| **CIDR** | **총 주소** | **AWS 가용 IP** | **쓰임** |
|---|---|---|---|
| `/16` | 2^16 = 65,536 | ~65,531 | VPC 전체 |
| `/20` | 2^12 = 4,096 | 4,091 | AZ별 블록 |
| `/24` (C클래스) | 2^8 = 256 | **251** | 티어별 서브넷 |
| `/28` | 2^4 = 16 | 11 | 최소 단위 |

- 서브넷 배치

| **티어** | **AZ 2a 블록 `10.xx.xx.xx/20`** | **AZ 2c 블록 `10.xx.xx.xx/20`** | **가용 IP** | **들어갈 것** |
|---|---|---|---|---|
| 퍼블릭 | 10.xx.xx.xx/24 | 10.xx.xx.xx/24 | 각 251 | External ALB, NAT GW ⚠️ |
| 프라이빗-앱 | 10.xx.xx.xx/24 | 10.xx.xx.xx/24 | 각 251 | SpringBoot(EC2 or ECS), Internal ALB, ~~Next.js~~ |
| 프라이빗-DB | 10.xx.xx.xx/24 | 10.xx.xx.xx/24 | 각 251 | A안: RDS MySQL+Redis / B안: VPN GW |

### 4. **퍼블릭 / 프라이빗 / 프라이빗-DB 분리**

- 티어의 정체성은 결국 **라우팅 테이블**이 정한다
  - `0.0.0.0/0 → IGW`면 퍼블릭
  - NAT만 있으면 프라이빗-앱
  - 인터넷 경로가 아예 없으면 프라이빗-DB.

| **티어** | **스택 / 거주 리소스** | **인터넷 방향** | **왜 분리하나** |
|---|---|---|---|
| 퍼블릭 | External ALB, NAT GW<br>만약 Next.js 서빙한다면 (앞단) CloudFront+S3로 서빙 | IGW 양방향 | 인터넷에 직접 닿아야 하는 것만<br>공격 표면 최소화 |
| 프라이빗-앱 | Spring Boot API(EC2/ECS), Internal ALB, Next.js SSR | NAT로 나가기 | 앱 서버에 공인IP X<br>이유 : 외부에서 직접 접근 X(노출 최소화) |
| 프라이빗-DB | **MySQL**(RDS 또는 VPN→온프렘), Redis | 완전 격리 | 개인정보(PII)·상태 저장소. 인터넷 경로 자체 없음. <br>앱에서만 접근. |

### **5. IGW & 라우팅 테이블**

**IGW는 VPC당 1개.**

참고 : 퍼블릭 RT가 `0.0.0.0/0 → IGW`를 가질 때만 인터넷 양방향이 열린다.

| **라우팅 테이블** | **대상 → 타깃** | **의미** |
|---|---|---|
| **퍼블릭 RT** | `10.xx.xx.xx/16 → local`<br>`0.0.0.0/0 → IGW` | 인터넷 양방향. ALB·NAT만 거주. |
| **앱 RT (2a)** | `local`<br>`0.0.0.0/0 → NAT-a(향후)`<br>`10.xx.xx.xx/16 → pcx` | 인입 불가, egress는 같은 AZ NAT. 피어링 시 경로 추가. |
| **앱 RT (2c)** | `local`<br>`0.0.0.0/0 → NAT-c` | AZ별 NAT로 포트고갈·장애 격리. |
| **데이터 RT** | A안: `local` 만<br>B안: `local` + `온프렘 CIDR → VGW` | A안 완전 격리 / B안은 VPN 경로만 추가(인터넷은 여전히 없음). |

### 6. **접근 제어 — Security Group 체인 + NACL**

- SG (Security Group)
  - 인터넷 인입은 **80/443**만
  - 앱 포트 **8080**(Spring Boot)은 ALB에서만
  - DB 포트 **3306**(MySQL)·**6379**(Redis)는 앱에서만
  - SSH(22)는 0.0.0.0/0 개방 금지 — **SSM Session Manager** 사용해보기

| **SG** | **인바운드 (from)** | **아웃바운드** |
|---|---|---|
| `alb-sg` (External ALB) | `80, 443 ← 0.0.0.0/0` (또는 CloudFront 관리형 prefix list) | `app-sg` 8080 |
| `app-sg` (Spring Boot) | `8080 ← alb-sg` · `8080 ← internal-alb-sg` **(SG 참조)** | `443 → 0.0.0.0/0`(NAT), `data-sg`로 3306/6379 |
| `data-sg` (MySQL/Redis) | `3306, 6379 ← app-sg` **만** | 최소(기본 차단). B안일 때만 VPN 대역. |
| `bastion-sg` (선택) | `22 ← 회사 IP` 또는 **SSM 사용** | app-sg 22 |

- **NACL (서브넷 단위, stateless) — 2차 방어선**
  - **stateless**라 응답용 ⚠️**ephemeral 포트(1024–65535)**를 양방향으로 열어줘야 한다.

| **서브넷** | **인바운드** | **아웃바운드** |
|---|---|---|
| 퍼블릭 | `80, 443 ← 0.0.0.0/0`, ephemeral | `80,443`, ephemeral |
| 앱 | 8080 ← **퍼블릭 서브넷 CIDR**, ephemeral | DB 서브넷 + ephemeral |
| DB | 3306/6379 ← **앱 서브넷 CIDR** 만, 그 외 deny | 앱 서브넷 ephemeral |

### 7. **AZ 이중화 (가용영역 분리)**

물리적으로 분리된 데이터센터 **2곳 이상**에 분산. <br>한 AZ가 죽거나 용량이 부족해도 다른 AZ가 받는다.

### 8. **NAT Gateway 배치 & 외부 호출 경로**

- **NAT는 프라이빗 서브넷이 밖으로 나갈 때만 쓰는 단방향 출구.**
  - 들어오는 요청엔 NAT 불필요 : ALB로 들어와 앱이 응답하는 트래픽은 세션 유지로 그 경로로 되돌아간다.
  - 앱이 먼저 밖으로 나갈때만 NAT : 크롤링/PG/알림톡/라이브 설명회 API or zoom 호출
  - AZ별 1개씩(NAT-a, NAT-c) + 각 EIP⚠️. 앱 RT를 같은 AZ NAT로 → cross-AZ 요금 회피 + SPOF 제거. ⚠️
  - 포트 고갈주의 : NAT 1개에 몰리면 SNAT 포트가 마름 (주의)
  - 고정 egress IP(EIP) : 외부 파트너 IP 화이트리스트 대응, 차단 시 NAT/EIP 교체로 우회

| **방향** | **경로** |
|---|---|
| 웹(정적/ISR) | 사용자 → CloudFront → S3(Next.js 자산) — 앱·NAT 경유 안 함 |
| 인바운드(API/SSR) | 사용자 → CloudFront → IGW → ALB → Spring Boot → MySQL/Redis → 응답 역순 |
| 읽기/쓰기 | 쓰기 → MySQL **Primary(3306)** · 읽기 → **Read Replica(3306)** (8장 참고) |
| 아웃바운드 | Spring Boot → 같은 AZ NAT → IGW → 인터넷 (크롤링·PG·Zoom) |

### **9. 데이터(DB) 티어 — MySQL · A안 / B안**

- **A안(RDS)**

  ![](/assets/img/2026-06-26/be-infra-3tier-vpc-design-1.png)

- **B안(온프레미스) 토폴로지**

  ![](/assets/img/2026-06-26/be-infra-3tier-vpc-design-2.png)

| 기준 | A안 — AWS RDS for MySQL | B안 — **온프렘 master/slave + VPN** |
|---|---|---|
| 위치 | DB 서브넷 안 RDS | AWS 밖. DB 서브넷은 VPN 게이트웨이 계층 |
| 읽기/쓰기 분리 | Primary(쓰기) + Read Replica(읽기) — 클릭으로 추가 | master(쓰기) + slave(읽기) — 직접 복제 구성 |
| 가용성(HA) | Multi-AZ 자동 failover, 자동 백업·PITR | 승격·백업 자체 책임(ProxySQL/MHA 등) |
| 운영 부담 | 낮음(패치·모니터링 관리형) | 높음(OS·MySQL·복제·VPN 직접) |
| 지연 | 같은 리전·AZ ms | VPN 홉 추가 → 지연·변동 ↑ |
| 비용 | 인스턴스+스토리지 시간당(예측 쉬움) | 자체 서버 + VPN/전송요금(기존 자산이면 절감) |
| 적합 | 신규 그린필드(본 프로젝트) | 기존 DB 자산·데이터 위치 고정이 필요할 때 |

- 궁금한 점<br>1번 Onpremise - 회사?<br>2번 Onpremise - 집?<br>⇒ 이게 나을까요 (Master/Slave는 분리되어야 하니까요)

### **10. 기타(프론트엔드 배포)**

| **후보** | **과금 구조** | **SSR** | **장단점** |
|---|---|---|---|
| **Vercel Pro** | $20/개발자·월 + 대역 1TB 포함, 초과 **$0.15/GB** (사용량 일부 $20 크레딧 상쇄) | 네이티브 | 최고의 DX·간편. 별도 벤더 + **Vercel→AWS API 교차 클라우드 전송**. 상용은 무료(Hobby) 불가. |
| **AWS Amplify Hosting** | 빌드 $0.01/분(1000 무료), 전송 **$0.15/GB**(15GB 무료), 저장 $0.023/GB, SSR 50만 요청+100GB-h 무료 후 $0.30/100만·$0.20/GB-h | 지원 | Vercel류 편의 + AWS 통합(per-seat 없음). 전송 단가는 Vercel과 같음. |
| ~~**CloudFront + S3** (정적/ISR export)~~ | S3 저장 $0.023/GB + CloudFront 전송 약 **$0.085/GB**(매월 1TB 무료 티어) | 불가(정적만) | **전송 단가 최저.** SEO 페이지·계산기 등 정적 위주에 최적. SSR 필요 페이지엔 부적합. |
| **같은 ECS/EC2 + CloudFront** (SSR 셀프호스팅) | 한계비용 ≈ 컨테이너 1개(이미 운영 중인 클러스터에 추가) | 가능 | 한 VPC·한 청구서, **교차 전송 0**. 운영은 직접. 비용 효율 최고. |

### 11. **Terraform**
