---
title: "AWS Domain & ACM 설정"
date: 2026-06-26 17:35:00 +0900
categories: [aws, network]
tags: [aws, route53, acm]
---

> 실제 서비스명, 도메인, 리소스 식별자 등은 보안을 위해 예시 값으로 바꿨습니다.
{: .prompt-info }

### 인증서 ↔ 도메인

- `*.app.example.com`
  - 연결되어있는 loadbalencer : my-admin-alb, my-main-alb

- `*.example.com`
  - 리소스 연결 없음

### **도메인 구조 (ACM이랑 엮어서)**

| **호스팅 영역 이름** | **유형** | **생성자** | **레코드 수** | **설명** |
|---|---|---|---|---|
| example-1.com | 퍼블릭 | Route 53 | 2 | HostedZone created by Route53 Registrar |
| example-2.com | 퍼블릭 | Route 53 | 2 | - |
| **example-3.com** | **퍼블릭** | **Route 53** | **5** | **HostedZone created by Route53 Registrar** |
| **example.com** | **퍼블릭** | **Route 53** | **23** | **HostedZone created by Route53 Registrar** |
| example-4.com | 퍼블릭 | Route 53 | 4 | - |
| example-5.com | 퍼블릭 | Route 53 | 2 | - |
| example-6.com | 퍼블릭 | Route 53 | 2 | - |

**서비스 C**

| **레코드 이름** | **유형** | **라우팅 정책** | **차별화 요소** | **별칭** | **값/트래픽 라우팅 대상** | **TTL(초)** |
|---|---|---|---|---|---|---|
| example-3.com | A | 단순 | - | 아니요 | x.x.x.x<br>x.x.x.x | 300 |
| example-3.com | NS | 단순 | - | 아니요 | ns-xxxx.awsdns-xx.org.<br>ns-xxx.awsdns-xx.net.<br>ns-xxxx.awsdns-xx.co.uk.<br>ns-xxx.awsdns-xx.com. | 172800 |
| example-3.com | SOA | 단순 | - | 아니요 | ns-xxxx.awsdns-xx.org. awsdns-hostmaster.amazon.com. 1 7200 900 1209600 86400 | 900 |
| example-3.com | TXT | 단순 | - | 아니요 | "google-site-verification=&lt;redacted&gt;" | 300 |
| www.example-3.com | CNAME | 단순 | - | 아니요 | sites.framer.app | 300 |

### **AMI (기존 사용중인 이미지)**
