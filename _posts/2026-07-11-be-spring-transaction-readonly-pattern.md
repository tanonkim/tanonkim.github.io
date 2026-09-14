---
title: "Transaction (서비스 A 적용기)"
date: 2026-07-11 10:46:00 +0900
categories: [spring, transaction]
tags: [spring, transaction]
---

> 실제 서비스명, 도메인, 리소스 식별자 등은 보안을 위해 예시 값으로 바꿨습니다.
{: .prompt-info }

### 무엇을 / 왜

읽기 서비스와 쓰기 서비스의 트랜잭션 경계가 올바른지 정적 점검. 목적:

- 읽기(readOnly=true)와 쓰기 분리 — 정합성 + 성능(읽기 최적화).
- **나중에 읽기/쓰기 데이터소스를 나눌 때** 이 애너테이션 정합성이 곧 라우팅 정확도가 되므로 미리 맞춰둠.

### 점검 결과 — ✅ 양호

| 점검 항목 | 결과 |
|---|---|
| QueryService 25개 중 readOnly 누락 | 0 |
| QueryService에 쓰기 트랜잭션 혼입 | 0 |
| CommandService 23개: 클래스 readOnly 기본 + 쓰기 오버라이드 | 23/23 |
| org.springframework.Transactional 사용(jakarta 혼용) | 혼용 X |

### 패턴 (서비스 A 규약)

- 쿼리 서비스 = 전부 읽기전용:

```java
  @Service
  @Transactional(readOnly = true)      // 클래스 전체 읽기전용
  public class ItemQueryService {
      public ItemResult get(Long id) { ... }   // 읽기만
  }
```

- 커맨드 서비스 = 클래스 readOnly 기본 + 쓰기 메서드만 오버라이드:

```java
  @Service
  @Transactional(readOnly = true)      // 기본은 읽기전용
  public class ItemCommandService {

      @Transactional                    // ← 쓰기 메서드만 오버라이드
      public Long create(CreateItemCommand cmd) { ... }

      @Transactional
      public void update(...) { ... }
      // 조회성 헬퍼는 클래스 기본(readOnly) 그대로
  }
```
