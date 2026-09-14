---
title: "사내 프로젝트 내부 구성"
date: 2026-05-16 22:21:00 +0900
categories: [cs, project]
tags: [architecture, project]
---

> 실제 서비스명, 도메인, 리소스 식별자 등은 보안을 위해 예시 값으로 바꿨습니다.
{: .prompt-info }

### **1. I사 회사 소개**

I사는 이커머스 판매자를 위한 전환율 최적화(CRO, Conversion Rate Optimization) SaaS 플랫폼을 개발하는 스타트업입니다.<br>메인 제품은 서비스 C로, 쇼핑몰에 방문한 고객이 이탈하지 않고 구매까지 이어지도록 자동화된 루프를 제공합니다.

**핵심 가치 제안**

- **Zero-Config 철학**: 스크립트 태그 한 줄만 삽입하면 데이터 수집·위젯 노출이 자동 시작
- **실시간 위젯**: 방문자 행동 패턴을 분석해 이탈 직전에 개인화 쿠폰·혜택 배너를 노출
- **AI 리포트**: LLM 파이프라인으로 마케팅 인사이트를 자동 생성
- **Sankey 분석**: 방문부터 구매까지 퍼널 시각화

### 2. 현재 프로젝트 아키텍처

**2-1. 전체 구성도**

```sql
[ 쇼핑몰 (Host Page) ]
       │
       ├── Loader (Bastion-lite)   ← S3/CDN에서 스크립트 로드
       ├── Logger (데이터 수집)
       └── Widget Iframe (UI)
               │ HTTPS/JSON
               ▼
[ Main Server (Node.js / Fastify) ] ←→ [ Redis (Cache/Queue) ]
       │                                        │ BullMQ
       ▼                                        ▼
[ MongoDB (Replica Set) ] ←──── [ Admin Server (Worker) ]
       ▲                              커머스 플랫폼 OAuth, LLM 리포트
       │
       ▼
[ Admin Frontend (Next.js 15) ]
```

**2-2. 프로젝트 역할**

| 프로젝트 | 역할 | 기술 스택 |
| --- | --- | --- |
| 메인 서버 | Main Server (API + Worker) | Node.js, Fastify, MongoDB, BullMQ |
| 어드민 서버 | Admin Server (분석 Worker + 커머스 플랫폼 연동) | Node.js, Fastify, BullMQ |
| 어드민 프론트엔드 | 판매자 대시보드 (Admin Frontend) | Next.js 15, React 19, TypeScript, Zustand |
| 위젯 | 쇼핑몰 삽입 스크립트 & 위젯 | Vanilla JS, Next.js 16 |

**2-3. 인프라 현황**

- **클라우드**: AWS Seoul (ap-northeast-2) 마이그레이션 완료
- **서버**: EC2 m7g Spot Instance (Graviton, Auto Scaling Group)
- **DB**: `온프레미스` MongoDB Replica Set (PSA 구성) + Redis
- **네트워킹**: Tailscale VPN(WireGuard)으로 온프레미스 ↔ AWS 연결
- **배포**: Admin Front는 Vercel, Backend는 ALB + ASG

### 3. 서비스 B 신규 프로젝트 구성 (진행예정)

3-1. 서비스 개요

**서비스 B** 는 유튜브 영상에 등장하는 상품을 발견하고, 여러 쇼핑몰의 가격을 비교한 뒤, 최종적으로 자사몰로 구매를 유도하는 **영상 기반 커머스 앱**입니다.

- 로직

```sql
[ 유튜브 영상 ]
      │
      │  영상 속 상품 인식
      ▼
[ 서비스 B 앱 (React Native 웹뷰) ]
      │
      ├── 여러 쇼핑몰 가격 비교 ──────────→ [ 서비스 C 연동 (최저가 확인) ]
      │
      └── 자사몰로 구매 유도 ──────────────→ [ 자사 쇼핑몰 ]
```

| 레이어 | 기술 | 역할 |
| --- | --- | --- |
| **모바일 앱** | React Native | 유저 앱 — 영상 탐색, 상품 발견, 가격 비교, 구매 |
| **백엔드** | Spring Boot 4 (Java) | API 서버 — 상품 데이터, 영상 분석, 쇼핑몰 연동, 서비스 C 연계 |
| **어드민** | Next.js | 운영 대시보드 — 상품 관리, 영상 매핑, 입점 쇼핑몰 관리 |
| **기존 연동** | 서비스 C(I사) | 최저가 비교 데이터 공급, 자사몰 전환율 최적화 |
