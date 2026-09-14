---
title: "시스템 디자인 인터뷰 - Twitter"
date: 2026-05-31 18:28:00 +0900
categories: [system-design, interview]
tags: [system-design, twitter, fan-out]
---

[System Design Interview Walkthrough: Design Twitter](https://www.youtube.com/watch?v=Nfa-uUHuFHg&list=PL5q3E8eRUieWtYLmRU3z94-vGRcwKr9tM&index=5)

1. 시스템의 요구사항을 정확하게 이해
   1. 면접관과 의견이 일치하는지
   2. 기능을 명확하게 정의

> Tip: 요구사항엔 5분 이상 쓰지 말 것 — 중요하지만 가장 흥미로운 부분이 아니고, 여기 시간을 너무 쓰는 게 흔한 실수.

## Step0

### 로드맵

Requirements → Core Entities → API or Interface → Data Flow → Design → Deep Dives

## Step1

### Functional Requirements (핵심 기능)

계정 생성 로그인

트윗 CRUD

Follow

Following 타임라인 보기

좋아요, 답글(reply), 리트윗

트윗 검색

### Non-Functional Requirements (품질)

수억명 scalability

tweet, likes, retweet 높은 volume 컨트롤

높은 가용성(availability) — 99.99% uptime

보안/프라이버시

low latency

![](/assets/img/2026-05-31/be-system-design-twitter-1.png)

## Step2

client(웹앱 + 모바일 iOS/Android) → load balancer → API gateway → 각 서비스

- 로드밸런서 : 트래픽을 여러 서버로 분산 (거의 모든 SD 인터뷰의 핵심). 논의 포인트 2가지
  - ①라우팅 알고리즘 : Round Robin(고르게 순환) / Least Connections(연결 최소 서버) / IP Hash(같은 IP→같은 서버) → Twitter는 Round Robin (단순·공정, 영속연결 불필요, stateless 백엔드)
  - ②OSI 계층 : L4(전송, TCP) vs L7(앱, HTTP/HTTPS, URL·헤더 등 콘텐츠 기반 라우팅 가능) → Twitter는 L7 (콘텐츠 기반 라우팅·feature rollout)
- API Gateway : 마이크로서비스 아키텍처(작고 독립적인 서비스로 분리, 각자 특정 기능 담당), LB 요청을 적절한 서비스로 포워딩. Client+LB+API Gateway는 약 90% 인터뷰에 등장(시작점)
- API
  - tweet crud
  - reply crud
  - search
  - timeline
  - profile

- Tweet 저장 (Twitter 자체 NoSQL DB명은 "Manhattan", 우리는 오픈소스 대안 MongoDB. 트윗엔 복잡한 join 없음 → 문서 하나만 반환)
  - MongoDB -> 성능 효율성이 뛰어남 (읽기/쓰기 작업이 빠르고 빈번한 애플리케이션에 적합)
  - 문서 자체는 JSON(key-value) : 작성자·내용·메타데이터(타임스탬프·해시태그·멘션·위치)·미디어·리트윗·좋아요

- BlobStorage 예) s3
  - 대량의 비정형 데이터 저장에 최적·빠른 조회 (트윗 문서엔 미디어 '참조'만 저장, 본체는 S3)

- 시스템 과부하 X
  - → 속도 제한기 (트윗 생성 수 제한)
  - Rate Limiter
- 캐시
  - Redis, Memcached
  - 인기있는 트윗 캐싱
- CDN
  - 전세계 사용자의 속도 개선

Reply API 분리 이유

- 확장성 - 인기있는 tweet은 수천개의 답글 → 답글 서비스 완전히 분리
- 성능 - 답글 전부를 로딩하지 않고 트윗만 빠르게 조회 (스크롤 시 나머지 답글 추가 로딩)

답글을 트윗 ID로 인덱싱 → 트윗 조회 시 답글도 함께 번들로 fetch·캐싱 (따로 읽지 않음). 쓰기 경로엔 Rate Limiter(봇 방지). 답글 문서 ≈ 트윗 문서 + tweet ID 필드

- 검색속도
  - elastic search :
    - 분산 검색엔진 (대용량·저지연·고신뢰, 풀텍스트 검색에 적합)
    - 트윗 내용, 사용자 이름, 해시태그에 대한 역방향 index를 elastic search에 인덱싱
- CDC
  - 트윗 데이터는 CDC(Change Data capture)이라는 과정을 통해 elastic search로 전송됨 (데이터의 변경사항을 캡쳐)

---

- 타임라인 구성
  - 일반유저 → `Fan out on Write `채택 (※ 단순 `Fan out on Read`는 아래처럼 느려서 기각)
    - 타임라인 요청 시 사용자가 팔로우 하는 계정 목록을 쿼리
    - 해당 계정의 모든 트윗을 가져와 시간순으로 정렬 → 느림 (latency가 안좋음)
    - [Fan out on Write] ⇒ 새 트윗을 **메시지 큐**에 적재 (버퍼; 동시 대량 생성 완화)
    - ⇒ fan-out 워커가 큐에서 꺼내 작성자의 팔로워 목록 조회 → 각 팔로워 타임라인 캐시 맨 앞에 prepend (덕분에 읽기가 매우 빠름)
  - 메가 인플루언서 → Fan out on Read 하이브리드 (팔로워 수백만이면 write로 모든 캐시 갱신 시 과부하)
    - 유명인이 트윗을 올리면 타임라인을 즉시 업데이트 하는 대신 자신의 타임라인을 확인할 때까지 기다렸다가 계정의 최신 트윗을 캐시에서 가져와서 사용자의 피드에 통합하는 방식

- 프로필 서비스
  - 유저/팔로워 - SQL
  - 사용자 데이터에는 간단한 SQL 데이터베이스를 제공
    - 구조화된 사용자 속성에 효율적 쿼리
    - ACID트랜잭션 → 무결성
    - Join 등
  - 팔로워 연결관리  - `graph DB`
    - 추천시스템(새 팔로워 추천)·무결성 시스템의 기반
  - Auth 서비스 분리 : 인증/인가 전담 별도 서비스 (보안 강화·집중 유지보수·서드파티 연동 유연성)

- 보안
  1. 인증/권한
  2. 데이터 암호화 — 전송 중(in transit) HTTPS / 저장 시(at rest) 대부분 최신 DB가 네이티브 지원
  3. Rate Limiting — 트윗·답글 속도 제한 + DDoS 방지 위해 API Gateway에서 IP 기반 제한
  4. 입력값 검증(Input Validation) — 입력을 검증·새니타이즈하여 SQL injection·XSS(Cross-Site Scripting) 등 방지, 클라이언트·서버 양쪽

- 모니터링
  1. 시스템 상태 점검

     서비스 중단X, Grafana + Prometheus

  2. Logging — 모든 액션 기록(디버깅·보안 추적). ELK 스택 = Elasticsearch(저장)+Logstash(처리)+Kibana(시각화)
  3. Alerts
     1. 로그인 실패·트래픽 급증 등 → Alert Manager/PagerDuty + Prometheus 연동, email·Slack 통지

- Testing
  - Load Test - 신규 기능 도입 전 부하를 견디는지 확인 → 병목·잠재 실패 파악
  - Automated Testing - CI(Jenkins/GitHub Actions)로 변경마다 Unit Test(개별)+Integration Test(서비스 간 통신) 자동 실행
  - Backup + Recovery - 정기 백업 필수 + 복구 프로세스를 주기적으로 테스트
