---
title: "시스템 디자인 인터뷰 - TicketMaster"
date: 2026-04-19 21:21:00 +0900
categories: [system-design, interview]
tags: [system-design, ticketmaster]
---

[System Design Interview: Design Ticketmaster w/ a Ex-Meta Staff Engineer](https://www.youtube.com/watch?v=fhdPyoO6aXI)

요구사항 → 코어 엔티티 → API → HighLevel 디자인 설계 → 딥다이브

### 요구사항

기능적 요구사항 (Functional Requirements)

- 티켓 예약
- 이벤트 확인
- 이벤트 검색

비기능적 요구사항 (NON-Functional Requirements)

- 가용성 or 일관성 등 어떤걸 우선순위로 할건지
  - 티켓 : 일관성
  - 확인/검색 : 가용성
- `read >> write`
- 인기 이벤트로 인한 확장성

### 코어 엔티티

- 이벤트
- 장소
- 공연자
- 티켓

### API

REST API 사용

GET /event/:eventId → 이벤트, 장소, 공연자, 티켓[] (티켓목록)

GET /search?term={term}&location={location}&type={type}&date={date} → `partial<Event>`

POST /booking/reserve

- header : JWT \| SessionTOken
- body : {ticketId}  // userId는 JWT에 넣기, body에 X

**Q. 만약 가족석처럼 다좌석 동시 예약이라면**

ticketIds[]로 확장 고려

**Q. 네트워크 재시도 시 중복 예약 방지**

Idempotency-key 헤더 필요 (멱등성)

POST /booking/confirm

- header : JWT \| SessionTOken
- body : {ticketId, paymentDetails}

**Q. 동시성 제어 (수만명이 같은 좌석을 동시에 클릭 → DB row의 병목)**

대안 : Redis 분산락(Redlock) 또는 Redis에서 상태 관리

JPA라면 낙관적 락 VS 비관적락 선택 근거?

### HighLevel Design

MSA 아키텍쳐

티켓 : 좌석선택 → 티켓 구매 + 타이머(10분)

![](/assets/img/2026-04-19/be-system-design-ticketmaster-1.png)

1. 이벤트 CRUD
   1. client ↔ API GateWay() ↔ 이벤트 CRUD service ↔ DB (Event - id, venueId, name, tikecets[])
2. 검색
3. 예약
   1. 좌석선택
   2. 티켓구매 (reserve, confirm)
      1. (available, reserved, booked)

- 예약 안된상태로 10분이 초과된다면
  1. Ticket 엔티티에 reservedTimeStamp 필드 추가
  2. ~~크론잡 배치~~ → Redis (TTL, ID)
     1. Redis가 죽는다면?

        관점 : Redis를 SPOF로 두지 않는다

**Q. 결제 실패**

confirm API에서 결제 실패 시 좌석 롤백은 → Saga 패턴 or 보상 트랜잭션 필요

### 딥다이브

1. 지연검색 (Search API)

   ![](/assets/img/2026-04-19/be-system-design-ticketmaster-2.png)

   현재 조회는 전체 테이블 스캔을 하기 때문에 느리다. ⇒ ElasticSearch 도입

   CDC(데이터 변경 감지) → 이벤트스트림

   데이터 변경이 많다면 CDC → 큐/배치 등 활용

2. 빠른 검색 속도 - 캐싱
   1. Elastic Search - Node Query Caching
   2. redis
   3. CDN (API 캐싱, 30초-1분) - Client와 API 캐싱 사이

3. 트래픽 처리

   **A. 실시간 업데이트**

   많은 사람들이 티켓 구매 시 첫 데이터는 정확하겠지만, 1초/2초/1분/5분/10분이 될수록 데이터는 정확하지 않음 → 실제 데이터X → 구매할 수 없는 티켓에 접근 → 사용자 경험 저하 ⇒ **실시간 업데이트**

   1. `Long Polling` (HTTP요청을 보내고 서버가 응답할 수 있도록 30s-1m동안 열어둠) 구현 쉬움/인프라 불필요
   2. `SSE` (서버전송 이벤트 _ 서버→클라 단방향 스트림)
      1. 만약 30s-1m보다 더 오래 머무른다면
      2. 웹소켓과는 다른 개념

   **B. 가상 대기열**

   ![](/assets/img/2026-04-19/be-system-design-ticketmaster-3.png)

   대기열 - Redis Sorted Set사용

4. **DB - 샤딩**

   개념 정리 필요
