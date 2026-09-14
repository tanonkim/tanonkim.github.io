---
title: "스레드 경계를 넘는 Context 전파"
date: 2026-02-15 18:47:00 +0900
categories: [spring, async-thread]
tags: [threadlocal, security-context, mdc]
---

```java
인증 → 컨텍스트 저장 → 권한 판단 → 서비스 실행 → 로그 추적 → 응답
이 흐름이 “스레드 경계”를 넘을 때 어떻게 유지되는가 라는 관점에서 공부 할부분
```

Spring 기반 웹 애플리케이션에서 요청 하나가 처리되는 동안 여러 컨텍스트 정보(인증, 트랜잭션, MDC 로그 등)가 함께 흘러야 한다. 문제는 **비동기 호출, @Async, WebFlux, 가상 스레드** 등으로 스레드가 바뀌는 순간 이 정보들이 기본적으로 사라진다.

### 동기 요청 (단일 스레드)

```java
HTTP 요청 → FilterChain → DispatcherServlet → Controller → Service → Repository → 응답
```

동기 흐름에서는 전부 ThreadLocal로 유지하므로 (같은 스레드 == 같은 컨텍스트) 별도의 작업이 필요없다.

| 컨텍스트 | 저장 방식 | 비고 |
| --- | --- | --- |
| SecurityContext | SecurityContextHolder → 내부 ThreadLocal | 필터에서 set, 응답 후 clear |
| 트랜잭션 | TransactionSynchronizationManager → ThreadLocal | @Transactional 진입 시 바인딩 |
| MDC (로그 추적) | MDC (SLF4J) → `ThreadLocal<Map>` | 필터에서 traceId 세팅 |
| RequestAttributes | RequestContextHolder → ThreadLocal | 요청 스코프 빈 등 |

### 스레드 경계를 넘는 해법

1. @Async

   → 해법: SecurityContext 전파 모드 변경

   ```java
   // 애플리케이션 시작 시
   SecurityContextHolder.setStrategyName(
       SecurityContextHolder.MODE_INHERITABLETHREADLOCAL
   );
   ```

   실무에서는 INHERITABLETHREADLOCAL이 스레드 풀 재사용 시 오염 위험이 있어 DelegatingSecurityContextExecutor를 사용한다.

   **예시 코드**

   ```java
   @Configuration
   @EnableAsync
   public class AsyncConfig implements AsyncConfigurer {
   
       @Override
       public Executor getAsyncExecutor() {
           ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
           executor.setCorePoolSize(5);
           executor.initialize();
           // 핵심: SecurityContext를 자식 스레드로 복사
           return new DelegatingSecurityContextAsyncTaskExecutor(executor);
       }
   }
   ```

   **MDC도 자식 스레드로 넘기기**

   ```java
   public class ContextCopyTaskDecorator implements TaskDecorator {
       @Override
       public Runnable decorate(Runnable runnable) {
           // 부모 스레드에서 캡처
           RequestAttributes attrs = RequestContextHolder.getRequestAttributes();
           SecurityContext sc = SecurityContextHolder.getContext();
           Map<String, String> mdc = MDC.getCopyOfContextMap();
   
           return () -> {
               try {
                   // 자식 스레드에 주입
                   RequestContextHolder.setRequestAttributes(attrs);
                   SecurityContextHolder.setContext(sc);
                   if (mdc != null) MDC.setContextMap(mdc);
                   runnable.run();
               } finally {
                   // 반드시 정리 (스레드 풀 오염 방지)
                   SecurityContextHolder.clearContext();
                   MDC.clear();
                   RequestContextHolder.resetRequestAttributes();
               }
           };
       }
   }
   ```

1. CompletableFuture / 직접 스레드 풀 사용

   ```java
   // SecurityContext를 명시적으로 감싸기
   SecurityContext context = SecurityContextHolder.getContext();
   Map<String, String> mdcMap = MDC.getCopyOfContextMap();
   
   CompletableFuture.supplyAsync(() -> {
       SecurityContextHolder.setContext(context);
       if (mdcMap != null) MDC.setContextMap(mdcMap);
       try {
           return myService.process();
       } finally {
           SecurityContextHolder.clearContext();
           MDC.clear();
       }
   }, taskExecutor);
   ```

2. WebFlux

   WebFlux는 ThreadLocal을 사용하지 않는다. Reactor Context라는 별도의 매커니즘을 사용한다.

3. Virtual Thread (Java 21+)

   가상 스레드는 캐리어 스레드 위에서 수시로 마운트/언마운트되지만, **ThreadLocal은 가상 스레드 단위로 유지된다**. 따라서 기존 동기 코드의 ThreadLocal 기반 컨텍스트가 그대로 동작한다.

   주의점 : 가상 스레드 수백만개 생성 가능 → ThreadLocal에 무거운 객체를 넣으면 메모리 문제
