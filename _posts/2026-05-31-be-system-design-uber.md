---
title: "시스템 디자인 인터뷰 - Uber"
date: 2026-05-31 18:33:00 +0900
categories: [system-design, interview]
tags: [system-design, uber, geohash]
---

[Design Uber w/ a Ex-Meta Staff Engineer: System Design Interview breakdown](https://www.youtube.com/watch?v=lsKU38RKQSo)

## Step0

### 로드맵

Requirements → Core Entities → API → Design → Deep Dives(Non Functional을 만족)

## Step1

### 1. Functional Requirements

1. 사용자가 출발지, 목적지 → 예상 요금 확인

   여러 차종은 제외 (우버 X만 사용)

2. 예상 견적(요금·ETA)을 기준으로 차량을 호출 (매칭 요청)
3. 운전자는 요청을 수락/거절, 픽업/하차 장소로 이동

- out of scope
  - 차량 종류
  - 평점/리뷰 (드라이버·라이더)
  - 스케쥴

### 2. Non Functional Requirements

> 확장성, 가용성, 무결성

낮은 지연시간 (매칭이 중요, 1분 미만 or Fail)

매칭의 일관성  - 항상 한명의 드라이버와 매칭

매칭 외 부분의 높은 가용성 (매칭=일관성, 그 외=가용성 우선)

높은 처리량을 처리 (Peak Hour)

- out of scope
  - GDPR, privacy
  - 시스템 오류
  - 모니터링/로깅/알람. …
  - CI/CD

⇒ 대략적인 추정치는 X, 설계에 직접적인 영향을 미치는 경우에만 추정치O

## Step2

### 1. Core Entities

Ride

Driver

Rider

Location

### 2. API

Core Entities를 활용해서 Functional 요구사항 만족

POST /ride/fare-estimate → `Partial<Ride>`

```
{ 
	source, 
	destination 
}
```

PATCH /ride/request

```
{
	rideId
}
```

POST /location/update

```
{
	lat, 
	long
}
```

PATCH /ride/driver/accept

```
{
	rideId
	true/false
}
```

PATCH /ride/driver/update

```
{
	rideId
	status: 'pickedup', 'droppedoff'
}
```

## Step3

### HighLevel-Design

목표 : 시스템 기능의 핵심 기능을 충족

Rider/Driver → AWS API GateWay/LoadBalancing/Routing/SSL/Rate Limiting → 1. Ride Service(견적요청) - 3rd Party 앱 (맵 등) ↔ Primary DB

1. Ride 매칭 Service (드라이버 매칭 서비스)

notification service (API) ⇒ 파이어베이스 등(블랙박스)

![](/assets/img/2026-05-31/be-system-design-uber-1.png)

Primary DB

- Ride : id, riderId, driverId, fare, eta, source, destination, status …
- Driver : id, status(inride, offline, matched), …

LocationDB

- 운전자의 location 저장

→ API (Driver)

## Step4

### Deep Dives 1. DB 속도

매칭 알고리즘, 일관성, DB속도 향상 등등

LocationDB와 Location Service에서 드라이버 업데이트로 인해 발생하는 초당 트랜잭션 수를 처리할 수 있어야 함 (600만/ 활성 : 300만, 5초마다 상태 업데이트 → 60만개)

⇒ DB·서비스가 초당 약 60만(600k) TPS를 감당해야 함

PostgreSQL 등 SQL / 위도/경도로 쿼리 실행 (2차원) ⇒ PostgreSQL은 초당 2000~4000건의 트랜잭션만 처리 가능함 ⇒ 좋은 옵션 X

1. postgresql
   1. 근접 위치 검색을 효율적으로 수행해야 한다.
   2. 초당 약 4K(4,000) TPS 수준으로 줄여야 한다 (postgres 한계 2~4K에 맞춤).

   **⇒ 지리 공간 인덱스 도입**

**지리공간인덱스**

- 쿼드 트리 사용
- POSTGIS
  - 재귀적으로 4구역으로 나눔

    ![](/assets/img/2026-05-31/be-system-design-uber-2.png)

  - 파란 점 → 드라이버가 많음 ⇒ 재귀적으로 4등분 ⇒ 트리가 4가지로 나뉨 ⇒ 또 트리를 분기처리 함
- **단점** : 초당 트랜잭션
  - 상당한 지연 → DB 내의 데이터가 정확도가 떨어진다.
  - 쿼드 트리 → 전체 `인덱스`를 다시 그려야 함 (메모리 공간을 차지 많이 함)

메모리 데이터 저장소

- 높은 TPS 처리
- 예) REDIS
  - geoHashing
  - 쿼드트리랑 비슷하지만 인덱스가 필요 없음

  ![](/assets/img/2026-05-31/be-system-design-uber-3.png)

  재귀적으로 지역을 나눔, 정밀하게 계속 나눔

  줄이 길어질수록 더 정확, 줄이 짧을수록 정확도가 떨어짐 : 예) 2 → 20 → 200 → 2000 …

- **장점** :
  - 계산하기 쉽다.
  - 추가적인 데이터 저장이 필요 없음
  - 쓰기(위치 업데이트) 빈도가 매우 높을 때 유리 — 별도 인덱스 재구성이 없어 갱신 비용이 낮음
- 단점
  - 밀도가 고르지 않을 때는 별로

⇒ 큐를 만드는건 오버엔지니어링

### 2. 초당 60만개 → 어떻게 낮출까?

10,20초 등등

더 정교하다면 클라이언트는 더 정확

속도/주차 ⇒ 주차했다면 업데이트할 필요 없음

외딴 곳 → 정확도가 필요없다

⇒ 동적 위치 업데이트 : TPS가 크게 떨어질 것

### 3. 일관성

한 운행에 두명의 운전자에게 연결되지 않도록

1. 한 운전자 받기

   운전자 요청 → 수락하지 않는다면 다음 운전자 요청/5초 대기 → …. while루프

   하나의 서버(인스턴스)에서 구성되면 됨

   - 만약 아무 상태가 5초 초과로 없었다면 상태 변경이 필요

     크론작업이 1분마다 실행, 차이가 5초 → 최대 55초가 될 수 있음 ⇒ 분산Lock (Redis 등)을 사용

     ![](/assets/img/2026-05-31/be-system-design-uber-4.png)

2. 서버는 수평확장되어있음 (여러 서버)

   10만명이 모인다면 한번에 100건의 요청을 받을 수도 있음 ⇒ X : N개의 인스턴스간의 조정이 필요

   Driver 엔티티에 status(requested_요청 받음 등) + Atomic 수준으로

3. DynamoDB 사용 등

   테이블에 대해 TTL을 생성할 수 있음 → Redis가 필요없을수 있음

   Driver Lock엔티티 - DriverId

### 4. 동적 확장

![](/assets/img/2026-05-31/be-system-design-uber-5.png)

큐 도입 (차량호출 대기열)

대기열에서 순서대로 운전자/승객을 연결

→ 수요가 급증하더라도 처리가 가능

유의 : **FIFO큐**인 경우

만약 1번 승객이 시골, 2번이 도심 → 2번은 대기가 길어짐

결론 : 위치 또는 지역별로 대기열을 나눠야 함

---

## ✅ 보완 & 핵심 정리 (영상 기준)

> 위 노트에서 빠졌거나 더 강조하면 좋은 부분 정리. 면접에서 점수가 갈리는 포인트 위주.

### API 보안 & 표현

- 인증: `userId`를 **body에 넣지 않고** JWT/세션 토큰을 **헤더**에 담는다. body에 userId를 넣으면 타인 명의로 호출 가능 → 보안 취약.
- 시니어 이상은 API에 데이터 타입 명시 생략 (enum처럼 특별한 경우만 명시).
- `/ride/driver/update` 응답: 다음 목적지 lat/lng 반환(없으면 null). 픽업 완료 → 목적지 좌표, 하차 완료 → null.

### 🔑 위치 인덱스: 쿼드트리 vs 지오해시 (가장 중요)

> **트레이드오프 한 줄 요약**
>
> 쿼드트리 = 밀도 불균형에 강함 / 업데이트 빈도 높으면 불리(매 갱신마다 트리 재인덱싱).
>
> 지오해시 = 업데이트 빈도 높을 때 강함(문자열만 갱신, 별도 인덱스 X) / 밀도 불균형엔 상대적 약함.
>
> ⇒ Uber는 밀도는 불균형하지만 **쓰기 빈도가 압도적** → 지오해시 채택. (쓰기 빈도가 결정 요인)

- 쿼드트리 **K값**: 한 셀의 최대 포인트 수 임계값. 초과하면 그 셀만 다시 4분할. (노트의 '파란 점 많으면 4등분'이 이것)
- 지오해시: 밀도와 무관하게 동일 정밀도 격자로 균등 분할. base32 문자열이라 **prefix(앞 글자) 공유 = 같은 상위 구역** → prefix 비교로 근접 검색.
- Redis는 geohash 기본 지원, 처리량 약 **10만~100만 TPS** (vs PostgreSQL 2~4천 TPS).
- 참고: Uber는 실제로 정사각형 대신 **H3(육각형)** 사용. 정사각형은 중심→모서리/변 거리가 달라 왜곡 → 육각형이 더 원에 가까워 균일.

### 600k TPS 낮추기 = 동적 위치 업데이트

- 단순: 업데이트 주기 5초 → 10/15/20초 (정확도와 트레이드오프).
- 정교(클라이언트 적응형 전송):
  - **상태**: 운행 수락을 안 받는 중이면 불필요
  - **속도/주차**: 멈춰 있으면 불필요
  - **핫존 근접도**: 외딴 곳이면 빈도 낮춤
- ⇒ TPS 대폭 감소 (예: 60만 → 10만 이하).

### 일관성: 요구사항 2개로 분리해서 이해

1. **한 ride가 동시에 여러 드라이버에게 가지 않게** (쉬움): ride matching service 단일 인스턴스 내 while 루프로 순차 요청 → 5~10초 대기 → 거절/무응답 시 다음 드라이버. 단일 서버 내 처리라 조정 불필요.
2. **한 드라이버가 동시에 여러 요청을 받지 않게** (어려움): 인스턴스가 수평 확장 → 같은 최근접 드라이버에게 동시에 알림 가능(콘서트 직후 등). N개 인스턴스 간 **전역 조정** 필요.
   - Driver status에 `request_sent` 락 상태 + Atomic 갱신.
   - 문제: 무응답 시 상태가 영원히 잠김.
   - 크론잡(주기적 해제): Δ지연 발생(1분 주기면 최대 55초 잠김) → 비효율.
   - ✅ 더 나음: **`TTL 분산 락`**. Redis `SET driverId TTL=5s`(5초 후 자동 해제) 또는 DynamoDB row TTL. 요청 전 락 확인 → 잠겨 있으면 다음 드라이버.

- ⚠️ 노트에서 크론잡/분산락은 1번 아래 적혀 있지만 **논리적으로는 2번의 해결책**이다.
- Ticket Master의 'no double booking'과 동일 패턴.

### 큐(서지 처리) 보완

- ride request queue로 수요 급증 흡수. **지역별 파티션**으로 FIFO의 시골→도심 블로킹 방지.
- 추가 이점: ride matching service가 처리 중 죽어도 메시지가 ack되지 않아 **재처리**됨(유실 X). 매칭이 10초씩 걸려 죽을 수 있어 중요.

### 매칭 외 고가용성 (별도 deep dive)

- 각 서비스 수평 확장 + 자체 로드밸런서, DynamoDB/Redis로 사실상 무한 확장.
- **지역별로 전체 스택을 데이터센터 단위 분리 배포**(북동부/캘리포니아/시애틀 등) → 큐 파티셔닝·부하 분산이 자연스러워짐.
- user처럼 지역 이동 가능한 데이터는 각 DC에 read replica/복제.
