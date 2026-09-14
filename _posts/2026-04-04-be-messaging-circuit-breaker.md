---
title: "Circuit Breaker"
date: 2026-04-04 19:42:00 +0900
categories: [messaging, resilience]
tags: [circuit-breaker, resilience4j]
---

> 실제 서비스명, 도메인, 리소스 식별자 등은 보안을 위해 예시 값으로 바꿨습니다.
{: .prompt-info }

## Circuit Breaker란?

### 핵심 문제: 원격 호출의 취약점

이 프로젝트는 4개의 외부 API를 호출한다.

```
Solapi (SMS) → https://api.solapi.com/messages/v4/send
Nurigo (알림톡) → https://api.coolsms.co.kr
AWS SES (이메일) → AWS 엔드포인트
Fingerpush (푸시) → https://api.fingerpush.com/push/...
```

**외부 API 호출이 왜 위험한가?**

일반 메서드 호출은 빠르게 실패하지만, 네트워크 호출은 그렇지 않습니다:

- 응답이 올 때까지 **수십 초를 기다릴 수** 있습니다
- 그 사이에 새 요청이 계속 들어오면 → **Thread가 모두 점유**됩니다
- Thread 고갈 → **ActiveMQ Worker 처리 불가** → **전체 시스템 다운**

Fowler가 말하는 핵심:

> "If you have many callers on an unresponsive supplier, you can run out of critical resources leading to **cascading failures** across multiple systems."

---

### Circuit Breaker의 3가지 상태

```
정상 시작
    │
    ▼
┌─────────────────────────────────────────────────────────────────────┐
│  CLOSED (닫힘 = 정상 작동)                                           │
│  - 외부 API 호출을 그대로 통과시킴                                     │
│  - 실패 횟수 카운트                                                    │
│  - 실패가 임계값 초과 → OPEN으로 전환                                   │
└─────────────────────────────────────────────────────────────────────┘
    │ 실패 5번 (예시)
    ▼
┌─────────────────────────────────────────────────────────────────────┐
│  OPEN (열림 = 차단)                                                   │
│  - 외부 API를 아예 호출하지 않음                                        │
│  - 즉시 에러 반환 (Fail Fast)                                          │
│  - 일정 시간(예: 60초) 후 → HALF-OPEN으로 전환                         │
└─────────────────────────────────────────────────────────────────────┘
    │ 60초 경과
    ▼
┌─────────────────────────────────────────────────────────────────────┐
│  HALF-OPEN (반열림 = 복구 테스트)                                      │
│  - 시험적으로 1번만 외부 API 호출                                       │
│  - 성공 → CLOSED로 복귀                                                │
│  - 실패 → 다시 OPEN으로                                                │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 이 프로젝트에서 왜 필요한가?

### 현재 코드의 문제점

현재 각 Sender는 이렇게 구현되어 있습니다:

```java
// SolapiSender (현재)
try {
    ResponseEntity<String> response = restClient.post()
        .uri("/messages/v4/send")
        .body(request)
        .retrieve()
        .toEntity(String.class);
    // 성공 처리
} catch (Exception e) {
    log.error("SMS 발송 실패", e);
    return SendResult.failure(e.getMessage());  // 그냥 실패 처리
}
```

**문제 시나리오:**

1. Solapi API가 갑자기 응답을 안 함 (타임아웃 30초)
2. ActiveMQ에서 SMS 메시지가 계속 쏟아짐 (CRITICAL 5개, HIGH 10개...)
3. 각 Worker Thread가 30초씩 Solapi 응답 기다리며 블록됨
4. Thread Pool 고갈 → 이메일, 알림톡, 푸시도 처리 못 함
5. **전체 알림 시스템 마비**

### Circuit Breaker 적용 후

```
Solapi 장애 발생
    │
    ├── 5번 실패 감지 → Circuit OPEN
    │
    ├── 이후 SMS 요청: 즉시 실패 반환 (30초 대기 없음)
    │   └── fallback 채널로 즉시 라우팅 가능
    │
    ├── 60초 후 HALF-OPEN → 1번 테스트
    │
    └── 복구 확인 → CLOSED 복귀
```

---

## 이 프로젝트에 적용하는 방법

Spring Boot에서 Circuit Breaker는 **Resilience4j**를 사용하는 것이 표준입니다.

### 1. 의존성 추가 (`build.gradle`)

```groovy
implementation 'io.github.resilience4j:resilience4j-spring-boot3:2.2.0'
implementation 'org.springframework.boot:spring-boot-starter-aop'  // AOP 필수
```

### 2. 설정 (`application.yml`)

```yaml
resilience4j:
  circuitbreaker:
    instances:
      solapi:                          # SMS용
        slidingWindowSize: 10          # 최근 10번 호출 기준
        failureRateThreshold: 50       # 50% 이상 실패 시 OPEN
        waitDurationInOpenState: 60s   # OPEN 후 60초 대기
        permittedNumberOfCallsInHalfOpenState: 3  # HALF-OPEN에서 3번 테스트
        minimumNumberOfCalls: 5        # 최소 5번 호출 후 판단

      nurigo:                          # 알림톡용
        slidingWindowSize: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 60s

      ses:                             # 이메일용
        slidingWindowSize: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 30s   # AWS는 빠르게 복구되는 경우 많음

      fingerpush:                      # 푸시용
        slidingWindowSize: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 60s
```

### 3. Sender에 적용 (예: `SolapiSender`)

```java
@Component
@RequiredArgsConstructor
public class SolapiSender implements NotificationSender {

    private final RestClient restClient;

    @CircuitBreaker(name = "solapi", fallbackMethod = "fallback")
    public SendResult send(String recipient, String content) {
        // 기존 코드 그대로 유지
        ResponseEntity<String> response = restClient.post()
            .uri("/messages/v4/send")
            .body(buildRequest(recipient, content))
            .retrieve()
            .toEntity(String.class);

        return SendResult.success(extractMessageId(response));
    }

    // Circuit이 OPEN 상태일 때 이 메서드가 호출됨
    private SendResult fallback(String recipient, String content, CallNotPermittedException e) {
        log.warn("SMS Circuit Breaker OPEN - Solapi 호출 차단됨: {}", e.getMessage());
        return SendResult.failure("Circuit Breaker OPEN: SMS 서비스 일시 중단");
    }
}
```

### 4. 이미 있는 fallback 채널과 자연스럽게 연동

이 프로젝트에는 이미 `fallbackChannel`이 있습니다. Circuit Breaker가 즉시 실패를 반환하면 **AbstractWorker의 기존 fallback 로직**이 그대로 동작합니다:

```
Circuit OPEN → SendResult.failure() 즉시 반환
    │
    └── AbstractWorker.handleFailure() 실행
            │
            ├── retryCount < maxRetry → 재시도 큐에 적재
            │   (Circuit 회복 후 성공 기대)
            │
            └── 최대 재시도 초과 → fallbackChannel로 라우팅
                (알림톡 실패 시 SMS로)
```

---

## 정리

| 구분 | 현재 (없음) | Circuit Breaker 적용 후 |
| --- | --- | --- |
| Solapi 장애 시 | Thread 30초씩 블록 | 즉시 실패 반환 |
| 장애 전파 | 전체 Worker 마비 | SMS만 영향, 나머지 정상 |
| 복구 감지 | 수동 확인 | 자동 HALF-OPEN 테스트 |
| fallback | 최대 재시도 후 작동 | Circuit OPEN 즉시 작동 |

이 프로젝트에서 Circuit Breaker를 적용할 가장 적합한 지점은 **각 Sender 클래스**(`SolapiSender`, `NurigoSender`, `SesV2Sender`, `FingerpushSender`)입니다. 기존 retry/fallback 로직은 건드릴 필요 없이 자연스럽게 연동됩니다.
