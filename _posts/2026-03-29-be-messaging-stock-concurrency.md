---
title: "재고관리 시스템으로 알아보는 동시성 이슈"
date: 2026-03-29 19:05:00 +0900
categories: [messaging, concurrency]
tags: [concurrency, redis, lock]
---

## 01. 문제 정의

```
재고 10개
동시에 20명 요청
→ 재고 -10 발생
```

👉 문제: overselling

---

## 02. race condition 이해

- 조회 + 수정은 atomic 하지 않음

---

## 03. synchronized

### 결과

- 단일 서버 OK

### 한계

- 서버 2대 이상 → 실패

---

## 04. DB Lock

### Pessimistic Lock

- 정확성 확보

### 문제

- 느림
- 트래픽 증가 시 DB 병목

---

## 05. Redis 분산 락

### 적용

- Redisson

### 장점

- 분산 환경 해결

### 단점

- 락 비용 존재

---

## 06. Redis Atomic 연산 (핵심)

### 적용

```
DECR stock
```

### 장점

- atomic 보장
- 빠름

### 핵심

- 락 없이 해결

👉 깨달음 →  "동시성은 연산으로 해결하는 게 최적"

---

## 07. 최종 구조

```
Redis → 재고 감소
DB → 저장
```

---

## 08. 핵심 요약

- 동시성 해결 → Redis
- DB는 결과 저장만

---

## 09. 내가 배운 것

- 동시성 문제는 반드시 발생한다
- 락은 성능을 죽인다
- Redis atomic이 가장 효율적이다

---

# 🧾 PR 정리 (Kafka 제거)

---

## PR 1. 기본 재고 차감

- DB 감소 로직

문제

- race condition 발생

---

## PR 2. synchronized

- JVM 락 적용

결과

- 단일 서버 OK

한계

- 분산 환경 실패

---

## PR 3. DB Lock

- Pessimistic Lock / Optimistic Lock / Named Lock

결과

- 정합성 확보

문제

- 성능 저하

Pessimistic Lock  
Optimistic Lock : 충돌이 빈번하게 나지 않을 때 (실제 Lock 걸지 않음, 재시도 로직 필요  
NamedLock : 분산 락을 구현할 때 주로 사용(세션 관리를 잘 해야 함)

---

## PR 4. Redis 분산 락

- Redisson

결과

- 분산 환경 해결

문제

- 비용 발생

---

## PR 5. Redis Atomic 연산 (최종)

- DECR

결과

- 성능 + 정확성 모두 확보
