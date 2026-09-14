---
title: "Virtual Thread"
date: 2026-02-12 00:18:00 +0900
categories: [spring, async-thread]
tags: [java, virtual-thread, spring-boot]
---

### **Virtual Thread란?**

기존 Java Platform Thread는 OS Thread와 1:1로 매핑되어 생성비용이 크고 수가 제한적이었다. ↔ Virtual Thread는 JVM이 관리하는 경량 스레드로, 수십~수백만개를 생성해도 부담이 적다. 특히 I/O 대기시간이 많은 작업에서 처리량을 크게 개선할 수 있다.

### SpringBoot 설정

Spring Boot 3.2부터 Virtual Thread를 공식 지원한다.. 설정은 매우 간단하다.

```java
# application.yml
spring:
  threads:
    virtual:
      enabled: true
```

이 한 줄로 Tomcat의 요청처리, @Async 작업, Spring MVC의 비동기 태스크 등에서 Virtual Thread를 사용하게 된다.

### 어디에 적용되나요

`spring.threads.virtual.enabled=true` 설정 시 다음 영역에 자동 된다.

- Tomcat/Jetty/Undertow 요청 처리 스레드
- @Async 메서드 실행
- Spring for Apache Kafka / RabbitMQ 리스너
- Spring Scheduling (`@Scheduled`)

```java
// ExecutorService로 직접 사용
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> {
        // 각 태스크가 Virtual Thread에서 실행
        callExternalApi();
    });
}

// Bean으로 등록
@Bean
public AsyncTaskExecutor applicationTaskExecutor() {
    return new TaskExecutorAdapter(Executors.newVirtualThreadPerTaskExecutor());
}
```

### Virtual Thread가 빛나는 경우

- DB 쿼리, HTTP 호출, 파일 I/O 등 **I/O 바운드 작업**이 많은 서비스
- 동시 요청 수가 많지만 각 요청이 대부분 대기 시간인 경우
- 기존 스레드 풀 크기 튜닝에 어려움을 겪던 경우

### 주의사항

- CPU 바운드 작업에 큰 이점이 없다.
- synchronized 블록 내에서 blocking I/O를 하면 thread가 pinning(고정)되어 성능이 저하된다.
- ThreadLocal 남용에 주의해야 한다. Virtual Thread는 매우 많이 생성되므로 무거운 객체를 저장하면 메모리 문제가 발생할 수 있다.
- 스레드 풀 크기 기반으로 동시성을 제어하던 패턴이 무효화된다.

### WebFlux와의 비교

|  | Virtual Thread + MVC | WebFlux (Reactive) |
| --- | --- | --- |
| 프로그래밍 모델 | 동기식 코드 | Mono/Flux 리액티브 스트림 |
| 학습곡선 | 낮음 | 높음 |
| 디버깅 | 쉬움 (일반 스택 트레이스) | 어려움 |
| 성능 | I/O 바운드에서 우수 | I/O 바운드에서 우수 |

Virtual Thread의 가장 큰 장점은 **기존 동기식 코드를 거의 변경하지 않고도** `높은 동시성`을 확보할 수 있다는 점이다. WebFlux처럼 리액티브 패러다임으로 전환할 필요 없이, 익숙한 Spring MVC 스타일을 유지하면서 성능을 개선할 수 있다.
