---
title: "Kafka LifeCycle With 배포전략"
date: 2026-04-16 19:26:00 +0900
categories: [messaging, kafka]
tags: [kafka, spring-kafka, deployment]
---

### 원본 코드

```java
@Service
public class CouponConsumerLifecycleService {

    private final Logger logger = LoggerFactory.getLogger(CouponConsumerLifecycleService.class);

    private final KafkaListenerEndpointRegistry registry;

    public CouponConsumerLifecycleService(KafkaListenerEndpointRegistry registry) {
        this.registry = registry;
    }

    // 앱 준비 완료 후 컨슈머 시작
    @EventListener(ApplicationReadyEvent.class)
    public void startConsumer() {
        MessageListenerContainer container = registry.getListenerContainer("couponCreatedConsumer");
        if (container != null && !container.isRunning()) {
            container.start();
            logger.info("couponCreatedConsumer started");
        }
    }

    // 필요 시 수동 정지 (점검, 배포 등) -> Controller 추가 or 특정 조건에서의 크론잡 사용
    public void stopConsumer() {
        MessageListenerContainer container = registry.getListenerContainer("couponCreatedConsumer");
        if (container != null && container.isRunning()) {
            container.stop();
            logger.info("couponCreatedConsumer stopped");
        }
    }


    public boolean isRunning() {
        MessageListenerContainer container = registry.getListenerContainer("couponCreatedConsumer");
        return container != null && container.isRunning();
    }
}
```

### stopConsumer()가 필요한 배포전략

어떤 배포일때 이런걸 써야하는지 누가 이걸 호출하게 만들어야하는지 정리를 해두면 더욱더 베스트 -> 블루-그린, 롤링 등일때

```java
    public void stopConsumer() {
        MessageListenerContainer container = registry.getListenerContainer("couponCreatedConsumer");
        if (container != null && container.isRunning()) {
            container.stop();
            logger.info("couponCreatedConsumer stopped");
        }
    }
```

| 배포 전략 | 언제 | 누가 |
| --- | --- | --- |
| 블루-그린 | 트래픽 전환 직전(구버전 중지) | CI/CD 스크립트 → 관리용 REST API |
| 롤링 | 인스턴스 종료 직전(SIGTERM) | @PreDestroy 훅 자동 호출 |
| 카나리 | 롤백 또는 이상 감지 즉시 | 모니터링 연동 or 운영자 수동 |

### 요약

1. 블루 그린이 가장 핵심적인 케이스인 이유
   - Green(신 버전)이 먼저 뜨면서 startConsumer()이 자동 호출
   - 이 시점에 Blue, Green 둘다 컨슈밍 → 쿠폰 중복 발급 위험
   - 트래픽 전환 전에 Blue의 stopConsumer()를 명시적으로 호출 필요
2. 롤링 배포 → @PreDestroy가 정답
   - 인스턴스 종료 시 JVM이 자동으로 호출
   - 쿠버네티스 preStop hook과 함께 쓰면 더 안전

## 배포 전략별 Kafka 라이프사이클 처리

### 1. 블루-그린 배포

```
[Load Balancer]
      │
  ┌───▼───┐     ┌───────┐
  │ Blue  │     │ Green │  ← 새 버전 (Standby)
  │(구버전) │     │(신버전)│
  │ LIVE  │     │ READY │
  └───────┘     └───────┘
```

- 흐름도
  1. Green(신버전) 인스턴스를 새로 띄움 → `ApplicationReadyEvent` 발생 → `startConsumer()` 자동 호출
  2. 이 시점에 Blue와 Green이 동시에 같은 토픽을 컨슘 → 중복 처리 위험
  3. 트래픽을 Green으로 전환하기 전, **Blue의 컨슈머를 먼저 정지**해야 함
  4. Blue의 `stopConsumer()` 호출 → Blue 컨슈머 정지 후 트래픽 전환

- stopConsumer() 호출 주체
  - 배포 스크립트(CI/CD) : 트래픽 전호나 직전에 Blue 서버의 관리용 API 호출
  - Actuator shutdown endpoint + Graceful Shutdown 설정
  - 운영자가 직접 관리용 REST API 호출

```java
// AdminController.java (내부망 전용 포트 or 인증 필수)
@PostMapping("/internal/consumer/stop")
public ResponseEntity<Void> stopConsumer() {
    lifecycleService.stopConsumer();
    return ResponseEntity.ok().build();
}
```

### 2. 롤링 배포

```
인스턴스 A: [구버전 LIVE] → [정지] → [신버전 LIVE]
인스턴스 B: [구버전 LIVE] → [구버전 LIVE] → [정지] → [신버전 LIVE]
인스턴스 C: ...
```

- 흐름
  1. 인스턴스를 하나씩 순차적으로 교체
  2. 인스턴스 종료 전 stopConsumer() → 현재 처리중인 메시지 완료 대기 → 종료
  3. 신버전 인스턴스 가동 → startConsumer() 자동 호출

핵심 고려사항 : Graceful Shutdown

Spring kafka의 graceful shutdown을 함께 설정해야 한다.

```yaml
# application.yml
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s  # 종료 대기 시간

  kafka:
    listener:
      ack-mode: MANUAL_IMMEDIATE  # 처리 완료 후 수동 커밋
```

- stopConsumer()의 호출 주체
  - `@PreDestroy` 훅: 컨테이너/프로세스 종료 신호(SIGTERM) 수신 시 자동 호출
  - Kubernetes의 `preStop` lifecycle hook

```java
// CouponConsumerLifecycleService.java 에 추가
@PreDestroy
public void onShutdown() {
    stopConsumer();
    logger.info("Consumer stopped on application shutdown");
}
```

### 카라니 배포

```
[Load Balancer]
   90%  │  10%
  ┌─────▼──┐  ┌──────▼──┐
  │ Stable │  │ Canary  │
  │ (구버전) │  │ (신버전) │
  └────────┘  └─────────┘
```

- 흐름
  1. 카나리(인스턴스) 인스턴스 일부만 띄움
  2. 같은 Consumer Group이면 (내부코드에선 group_1) kafka가 파티션을 자동으로 재분배
  3. 문제 발생 시 카나리 인스턴스의 stopConsumer() 호출 → 파티션이 Stable로 재할당

- stopConsumer()의 호출 주체
  - 모니터링 시스템 연동 (에러율 임계값 초과 시 자동 호출)
  - 운영자가 직접 관리용 API 호출 (롤백 시)

참고 : autoStartup = “false”를 사용하는 이유

```java
@KafkaListener(
            id = "couponCreatedConsumer", // 컨테이너 Id 부여
            topics = "coupon_create",
            groupId = "group_1",
            autoStartup = "false" // 이 컨슈머는 수동 시작
    )
    public void listener(Long userId) {
        try {
            couponRepository.save(new Coupon(userId));
        }
        catch (Exception e) {
            logger.error("failed to save coupon::" +  userId);
            failureEventRepository.save(new FailedEvent(userId));
        }
    }
```

Spring Context가 완전히 초기화되기 전에 kafka 메세지가 들어오면 DB connection 또는 Repository Bean이 준비되지 않아 처리 실패 가능성이 있다. → autoStartup = "false" + ApplicationReadyEvent를 사용하면 모든 Bean이 준비된 이후에 컨슈밍을 시작함 (문제 방지)
