---
title: "시스템 디자인 인터뷰 - Bitly"
date: 2026-05-31 18:39:00 +0900
categories: [system-design, interview]
tags: [system-design, url-shortener]
---

[Beginner System Design Interview: Design Bitly w/ a Ex-Meta Staff Engineer](https://www.youtube.com/watch?v=iUU4O1sWtJA)

## Step0

### 로드맵

Requirements → Core Entities → API or Interface → Data Flow → Design → Deep Dives

(※ Data Flow는 인프라 중심 문제에서만 필요 — 레이트리미터/메시지 큐 등. URL 단축기 같은 사용자 대면 제품은 생략)

## Step1

### Functional Requirements (핵심 기능)

URL Shortner : 긴 URL을 축소 → 입력 시 긴 URL로 이동

- 긴 url → short url을 생성
  - 별칭을 지정 (옵션)
  - 만료기간 설정(옵션)
- 단축 url에서 원래 url로 이동

### Non Functional Requirements (품질)

일관성↔가용성, 확장성, 호환성 등등 좋은 사용자 경험

- 리다이렉트 하는데 Low Latency 지연시간 (~200ms)
- 스케일 / 일일 활성 사용자(DAU) 1억명, 누적 10억개 URL 생성 (※ 10억은 하루치가 아니라 전체 기간 누적)
- **고유한** 단축코드
  - 실제로 사용하는게 시간이 필요하다.
  - 몇초~최대 1분까지 리다이렉트 필요 없음 ⇒ 1분후 다시 시도해주세요 문구 발행
- 높은 가용성 + 최종 일관성 (high availability, eventual consistency for URL shortening)
  - **CAP 정리**: 일관성(C)·가용성(A)·분할내성(P) 중 동시에 2개만 가능. 분산 시스템은 P(파티션)가 사실상 필수 → 결국 **C vs A** 중 선택
  - 핵심 질문 — 모든 읽기가 항상 '최신 쓰기'를 읽어야 하나? (강한 read-after-write 일관성 필요 여부)
    - 필요한 예: 은행 거래, 티켓 예매(마지막 좌석을 두 명이 동시에 예약하면 치명적)
    - URL 단축: 그럴 필요 없음 → **가용성**을 택하고 최종 일관성 허용
  - (바로 위 '고유한 단축코드' 항목의 두 메모가 최종 일관성을 택할 수 있는 근거: 단축 URL이 공유·사용되기까지 시간이 걸리고, 최악의 경우에도 "잠시 후 다시 시도" 안내로 충분)

BOTE (Back-of-the-envelope, 개략 추정)

- scale
- latency
- storage

계산 → 방향/결정에 직접적인 영향을 줄 때 하기, 바로 하는건 비추

## Step2

### Core Entities

- Original url
- Short url
- User

### API

클라이언트-시스템 간 계약

// shorten a url

- POST /urls → shortUrl

```
{
	originalUrl,
	alias?
	expirationTime?
}
```

// redirection

GET /{shortUrl} → Redirect to Original URL

## Step3

### HighLevel Design

client —getShortUrl(long), redirect(short)→ Primary Server(counter/create short url) → DB

- 생성 시: customAlias가 오면 DB에 이미 있는지 확인 후 없으면 그 별칭으로 저장
- 리다이렉트 시: 현재시각 > expirationTime이면 302 대신 만료 에러 반환

URL Entities

- shortUrl/customAlias
- longUrl
- expirationTime?
- createdAt
- userId

User

- userId
- …

302 Redirect : 해당 URL로 이동하여 자동으로 접속 (Temporary)

301 Redirect - Permanent

302 ⇒ 얼마나 자주 조회되는지 파악이 가능하다.

301 ⇒ 서버에 도달하지 않는다 (브라우저·DNS가 캐싱 → 컴퓨트 비용↓, 그러나 분석·로깅·동작 확인 불가)

→ 보통 302 권장: 사용자에게 분석을 노출하지 않더라도 내부적으로 트래픽/동작을 추적해 장애를 빨리 감지

## Step4

### DeepDives

- 단축코드의 고유성
  1. fast
  2. unique
  3. short (5~7자)

![](/assets/img/2026-05-31/be-system-design-bitly-1.png)

**[고유성]**

1. 긴 URL의 접두사를 사용 : 좋지 않음 ⇒ [www.facebook.com](http://www.facebook.com) → 같은 접두사 사용
2. 난수 생성기 : 누적 10억개 URL 생성(※ 하루가 아닌 전체 기간 누적) ⇒ 최소한 10억개 (10^9)
   1. base62 encoding 0-9, A-Z, a-z
   2. 62^6 = 56B (충돌 확률이 높음)
   3. Birthday Paradox ⇒ 10억건 중 88만개는 충돌 가능
   4. 대응: 저장 전에 DB에서 short code 존재 여부를 먼저 확인(read) → 없을 때만 저장 (읽기 1회 추가)
3. 긴 URL을 Hash (avalanche 효과로 출력이 난수와 사실상 동일 → 충돌 확률·DB 사전확인 필요성도 난수 방식과 동일)
   1. md5/SHA256… → Hash → base62(hash)[:6]
4. Counter
   1. url생성 시 counter 증가 → base62
      1. 560억개(62^6), 선형 증가라 충돌 없음 → 난수/해시와 달리 DB 사전 확인 불필요
   2. 예측가능하기에 보안에 좋지 않다.
      1. 개인 URL은 축약하지 마세요
      2. rate limiting
      3. biject function → 일대일 매칭하는 함수 (Sqids 같은 라이브러리 — 영상의 'squids'를 잘못 적은 듯, 예전 이름은 Hashids)

**[속도]**

- 인덱스
  - **고유성 보장 (PK에 자동으로 생성)**
  - 메모리에 저장된 인덱스를 사용 → 훨씬 더 빠름
    - Postgresql : BTree
    - Hash Index
- 캐시
  - Redis, MemCached
  - Read-Through + LRU 캐시 (※ '읽기 전용'이 아니라 '읽기 통과(Read-Through)' 방식)
    - 동작: 캐시 미스 시 DB에서 읽어 캐시에 채운 뒤 반환 → 이후 동일 요청은 O(1)
    - LRU 제거: 캐시가 가득 차면 오래 안 쓴 항목부터 제거 (오래된 URL은 점점 안 쓰임)
    - key=short code, value=long URL 형태
- CDN
  - 전 세계 엣지 서버에 캐싱해 지역 간 지연 감소
  - 단, 301과 동일한 단점: 요청이 우리 서버에 도달하지 않아 로깅·분석·동작 확인 불가 → 영상에서는 캐시까지만 쓰고 CDN은 생략 권장
  - 캘리포니아 서버, 중국요청 : 직접 요청은 너무 느림 →  CDN 사용

**[고가용성]**

- Scale Out (※ 이 '[고가용성]' 묶음은 사실 대부분 **확장성(Scale)** 주제 — 진짜 고가용성은 맨 아래 'DB 복제본·스냅샷' 부분)

초당 1000개의 요청 ⇒ EC2 T3 미디엄은 한번에 약 1000개의 요청 처리 가능 : 하나의 인스턴스

피크시간 → 그 이상

1만 ~ 10만(10K~100K, 피크 추정치) → Vertical Scale (서버의 성능을 확장) / Scale Out (여러 서버로 분산)

- MSA 아키텍쳐

Client ↔ API Gateway ↔ (Read Service / Write Service) ↔ DB

- 읽기(리다이렉트)는 많고 쓰기(생성)는 적음 → 읽기 서비스를 더 많이 수평 확장
- 각 서비스 앞에 로드밸런서(라운드로빈 등)가 요청 분배, 오토스케일은 클라우드가 처리
- 단, 코드가 단순해 두 서비스로 쪼개는 게 과할 수도 있음 (한 박스로 둔 채 수평확장해도 무방 — 트레이드오프)

20%미만 → 인스턴스 종료/75%이상이면 인스턴스 생성

- Write Service가 여러개라면 ⇒ Counter 분리
  - Counter : DB조회없이 고유성을 보장ㄴ

  ![](/assets/img/2026-05-31/be-system-design-bitly-2.png)

  Counter → `Redis의 incr 명령어 사용` (싱글쓰레드라 동시성 걱정 없음, 인메모리라 빠름)

  - 최적화: 서버가 켜질 때 카운터 1,000개를 한 번에 받아와 메모리에 두고 소진 → 매 요청마다 네트워크 호출 안 함 (서버 다운 시 남은 번호는 버려도 무방)

- DB의 확장 (고려사항)
  - 저장량 계산 먼저: 행당 ≈ 500 bytes × 10억 = 약 **500GB**
  - 500GB는 최신 SSD(수백 TB)엔 작은 양 → **단일 DB로 충분, 샤딩 불필요** (읽기도 대부분 Redis가 흡수)
  - 정말 필요해지면 그때 샤딩: short URL 기준 (예: hash(shortURL) % N)

- 고가용성

DB 복제본/스냅샷

- Redis 캐시가 죽어도 OK: read-through라 DB에서 다시 채움 (잠깐 느려질 뿐)
- 글로벌 카운터가 죽으면 문제 → 이중화(redundancy) + 주기적으로 카운트를 디스크에 스냅샷
- DB: 복제본(replica)을 두어 장애 시 승격, + 1시간 단위 스냅샷을 S3(저렴한 blob 저장소)에 백업

![](/assets/img/2026-05-31/be-system-design-bitly-3.png)
