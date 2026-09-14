---
title: "Kafka를 통한 선착순 쿠폰발행"
date: 2026-03-29 18:41:00 +0900
categories: [messaging, kafka]
tags: [kafka, redis, concurrency]
---

## 01. 문제 정의

### 목표

- 선착순 100명에게만 쿠폰 발급
- 동시 요청(수천 ~ 수만 TPS)에서도 정확성 보장

---

### 처음 접근

```
쿠폰 요청 → DB insert
```

---

### 문제 발생

- 동시 요청 시 100개 초과 발급
- race condition 발생
- DB가 동시성 제어를 못함

---

## 02. 동시성 문제 이해

### race condition

```
Thread A: count = 99
Thread B: count = 99

둘 다 insert 수행 → 101개 발급
```

---

### 해결이 필요한 것

- "딱 100개까지만" 보장해야 함
- 원자적(atomic) 연산 필요

---

## 03. Redis로 동시성 해결

### 적용 방법

```
Redis INCR 사용
```

---

### 흐름

```
1. Redis count 증가
2. 100 이하이면 통과
3. 초과하면 실패
```

---

### 장점

- atomic 연산 → race condition 해결
- 매우 빠른 처리 속도

---

### 한계

- API 서버가 여전히 DB까지 처리
- 트래픽 증가 시 서버 부하 존재

---

## 04. 동기 처리 구조의 한계

### 기존 구조

```
Client → API → Redis → DB
```

---

### 문제

- API 서버가 모든 작업 처리
- DB write가 병목
- 응답 시간 증가

---

### 깨달은 점

> "처리와 응답을 분리해야 한다"

---

## 05. Kafka 도입 (비동기 처리)

### 구조 변경

```
Client → API → Redis → Kafka → Consumer → DB
```

---

### 역할 변화

### API 서버

- 기존: 쿠폰 발급 직접 처리
- 변경: Kafka 메시지 발행

### Consumer 서버

- Kafka 메시지 소비
- DB 저장 처리

---

### 효과

- API 응답 속도 개선
- 트래픽 분산
- DB 부하 감소

---

### 핵심 개념

- 이벤트 기반 아키텍처 (Event Driven Architecture)
- 비동기 처리 (Async Processing)

---

## 06. 메시지 시스템 이해

### Kafka 특징

- at-least-once delivery
- 메시지 중복 가능

---

### 문제

```
같은 쿠폰 요청이 2번 처리될 수 있음
```

---

### 해결 필요

- idempotency (멱등성)

---

### 방법

- DB unique key
- Redis key
- 요청 ID 기반 처리

---

### 깨달은 점

> "비동기 시스템은 반드시 중복 처리를 고려해야 한다"

---

## 07. 최종 아키텍처 정리

```
Redis → 선착순 제어
Kafka → 비동기 처리
Consumer → DB 저장
```

---

### 특징

- 정확성 (100개 제한)
- 성능 (비동기 처리)
- 확장성 (Consumer scale-out)

---

## 08. 시스템 설계 핵심 요약

### Before

```
동기 처리 + DB 의존
```

---

### After

```
동시성 제어 + 비동기 처리 + 이벤트 기반
```
