---
title: "MDC와 ThreadLocal"
date: 2026-02-14 23:35:00 +0900
categories: [spring, async-thread]
tags: [mdc, threadlocal, logging]
---

## MDC

**로깅 프레임워크**가 제공하는 **스레드 단위의 키-값 저장소**

### **왜 필요한가**

서버가 동시에 수백개의 요청을 처리할 때 로그가 섞여서 특정 요청의 흐름을 추적하기가 어렵다. MDC를 사용하면 각 요청에 고유한 식별 정보를 심어두고, 모든 로그에 자동으로 출력할 수 있다.

### 사용 예

1. Filter에서 요청 ID를 세팅

```java
@Component
public class MdcFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(...) {
        try {
            MDC.put("requestId", UUID.randomUUID().toString());
            filterChain.doFilter(request, response);
        } finally {
            MDC.clear();
        }
    }
}
```

1. logback 패턴에 `%X{key}`추가

```java
<pattern>[%d] [%X{requestId}] [%thread] %-5level %logger - %msg%n</pattern>


// 출력 결과

[2026-02-14] [a3f2-b8c1] [http-nio-8080-1] INFO  UserService - 유저 조회 시작
[2026-02-14] [a3f2-b8c1] [http-nio-8080-1] INFO  UserRepository - DB 쿼리 실행
[2026-02-14] [d7e9-4a12] [http-nio-8080-2] INFO  UserService - 유저 조회 시작
```

## MDC와 ThreadLocal

MDC는 내부적으로 ThreadLocal로 구현되어있다.

SFL4J/Logback의 MDC는 `ThreadLocal<Map<String, String>>`을 사용해 **현재 스레드**에 키-값 쌍의 진단정보(requestId, userId 등등)을 저장한다. 그래서 같은 스레드 안에서는 어디서든 MDC.get(”key”)로 값을 꺼낼 수 있고, `%X{key}`로 자동 출력할 수 있다.

### 주의사항

1. **비동기 처리 시 전파안됨**

@Async, CompletableFuture, 새 스레드풀 등에서 MDC값이 자동으로 넘어가지 않는다. TaskDecorator를 구현해서 부모 스레드의 MDC 컨텍스트를 자식 스레드로 `복사`해야 한다.

```java
public class MdcTaskDecorator implements TaskDecorator {
    @Override
    public Runnable decorate(Runnable runnable) {
        Map<String, String> contextMap = MDC.getCopyOfContextMap();
        return () -> {
            try {
                if (contextMap != null) MDC.setContextMap(contextMap);
                runnable.run();
            } finally {
                MDC.clear();
            }
        };
    }
}
```

1. **스레드 풀 재사용 시 오염**

톰캣 등의 **스레드 풀**에서 스레드가 재사용되므로 요청 처리 후 MDC.clear()을 반드시 호출해야 한다. 보통 Filter, Interceptor의 afterCompletion에서 정리한다.

1. WebFlux(리액티브)에서 사용 불가

리액티브 환경은 하나의 요청이 여러 스레드를 넘나들기 때문에 ThreadLocal 기반인 MDC가 동작하지 않는다. 대신 Reactor Context를 사용해야 한다.

Spring에서 ThreadLocal을 다룰 때 겪는 문제(비동기 전파, 스레드 풀 오염, 리액티브 비호환)가 MDC에도 그대로 적용됩니다.
