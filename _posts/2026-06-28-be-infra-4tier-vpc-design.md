---
title: "4-Tier VPC / 네트워크 아키텍처 설계"
date: 2026-06-28 21:33:00 +0900
categories: [aws, vpc]
tags: [aws, vpc, alb]
---

> 실제 서비스명, 도메인, 리소스 식별자 등은 보안을 위해 예시 값으로 바꿨습니다.
{: .prompt-info }

3-Tier에서 Internal ALB를 더해 ALB를 퍼블릭/프라이빗 두 개로 분리한 구조이다.

단일 App(Spring Boot) - 웹/WAS로 나누지 않는다.

흐름 : 사용자 → public ALB → 내부 ALB(프라이빗 서브넷) → App(프라이빗) → DB(프라이빗)

> 보안상 이미지를 생략했습니다.

4계층 : Public ALB, 내부 ALB, DB<br>핵심은 ALB를 퍼블릭/프라이빗으로 분리한 것

1. 퍼블릭 ALB가 App을 타깃 + 내부도 ALB도 App을 타깃(내부 호출용) → Terraform으로 구성
2. 퍼블릭을 NLB → 내부 ALB로 직렬이 된다.

- 참고

|  | **ALB** (지금 사용) | **NLB** |
|---|---|---|
| 계층 | **L7** (애플리케이션, HTTP/HTTPS 이해) | **L4** (네트워크, TCP/UDP/TLS) |
| 라우팅 | URL 경로·호스트·헤더 기반 | IP:포트 기반 단순 전달 |
| 기능 | 경로 라우팅, 리다이렉트, WAF·인증 연동 | L7 기능 없음, 대신 **초저지연·초고처리량**(수백만 RPS) |
| IP | 동적(DNS) | **고정 IP/EIP 지정 가능** |
| 타깃 | instance·ip·lambda (다른 ALB는 **타깃 불가**) | instance·ip·**ALB를 타깃 가능** |

**서브넷 배치(4티어 X 2AZ = 8서브넷)**

| 티어 | AZ 2a | AZ 2c | 가용 IP | 들어갈 것 |
|---|---|---|---|---|
| Public | 10.xx.xx.xx/24 | 10.xx.xx.xx/24 | 각 251 | **퍼블릭 ALB**(External), NAT GW(+EIP) |
| Private-App | 10.xx.xx.xx/24 | 10.xx.xx.xx/24 | 각 251 | **내부 ALB**(Internal), App(Spring Boot) |
| DB | 10.xx.xx.xx/24 | 10.xx.xx.xx/24 | 각 251 | RDS MySQL, ElastiCache Redis |

**SG 체인**

| **SG** | **인바운드 (from)** | **아웃바운드** |
|---|---|---|
| `alb-sg` (External ALB) | `80, 443 ← 0.0.0.0/0` | `web-sg` 3000 |
| `web-sg` (Next.js) | `3000 ← alb-sg` **만** | `443→0.0.0.0/0`(NAT), `internal-alb-sg` |
| `internal-alb-sg` | `← web-sg` | `app-sg` 8080 |
| `app-sg` (Spring Boot) | `8080 ← internal-alb-sg` **만** | `443→0.0.0.0/0`(NAT), `data-sg` 3306/6379 |
| `data-sg` (MySQL/Redis) | `3306, 6379 ← app-sg` **만** | 최소(기본 차단) |

### **3-Tier vs 4-Tier**

| 관점 | 3-Tier | 4-Tier (ALB 퍼블릭/프라이빗 분리) |
|---|---|---|
| ALB | 1개(퍼블릭만) | 2개(퍼블릭 External + 프라이빗 Internal) |
| 계층 | Public · App · DB | 퍼블릭ALB · 내부ALB · App · DB |
| 컴퓨트 | 단일 App(Spring Boot) — 둘 다 동일 (web/WAS 분리 아님) | 외부 ALB → 웹 → Internal ALB → 앱 |
| 내부 진입점 | 없음(외부 ALB만) | 내부 ALB로 내부 호출 로드밸런싱·분리 |
| 서브넷 수 | 6 | 6 (동일) |
| 비용 | ALB 1개 | ALB 2개 (+$16~/월) |

![](/assets/img/2026-06-28/be-infra-4tier-vpc-design-1.png)

### Terraform

서비스 A Terraform
