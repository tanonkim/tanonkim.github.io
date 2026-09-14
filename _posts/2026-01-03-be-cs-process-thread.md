---
title: "프로세스/쓰레드"
date: 2026-01-03 17:39:00 +0900
categories: [cs, computer-architecture]
tags: [process, thread, spring]
---

프로세스와 쓰레드는 운영체제에서 작업을 실행하는 단위

### 프로세스

실행 중인 프로그램의 인스턴스(메모리에 할당된 객체)<br>독립적인 메모리 공간(코드/데이터/힙/스택)을 가지며 다른 프로세스와 메모리를 공유하지 않는다. 프로세스 간 통신(IPC)를 하려면 파이프,소켓,공유 메모리와 같은 매커니즘이 필요하다.

⇒ 생성비용이 높고 Context Switch(실행 중이던 프로세스/스레드가 다른 프로세스/스레드로 교체되는 것)가 느리다.

### 쓰레드

프로세스내에서 실행되는 실행 단위

프로세스 내의 쓰레드는 코드/데이터/힙 영역을 공유하고 각자 고유한 스택과 레지스터만 가진다.

### 멀티 프로세스

여러개의 독립적인 프로세스가 동시에 실행되는 방식

```java
[프로세스 A]           [프로세스 B]           [프로세스 C]
┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│ 코드         │      │ 코드         │      │ 코드         │
│ 데이터       │      │ 데이터        │      │ 데이터        │
│ 힙          │      │ 힙           │      │ 힙          │
│ 스택         │      │ 스택         │      │ 스택         │
└─────────────┘      └─────────────┘      └─────────────┘
=> 완전히 분리된 메모리 공간
```

### 멀티 쓰레드

하나의 프로세스 내에서 여러 쓰레드가 동시에 실행되는 방식

```java
[프로세스]
┌────────────────────────────────────┐
│ 코드 (공유)                          │
│ 데이터 (공유)                         │
│ 힙 (공유)                            │
├───────────┬───────────┬────────────┤
│ 스택 A     │ 스택 B     │ 스택 C      │
│ (스레드1)   │ (스레드2)  │ (스레드3)    │
└───────────┴───────────┴────────────┘
```

| 구분 | 멀티 프로세스 | 멀티 스레드 |
| --- | --- | --- |
| 메모리 | 각자 독립 | 힙/데이터 공유 |
| 통신 | IPC (느림) | 공유 변수 (빠름) |
| 동기화 | 상대적으로 단순 | 락, 세마포어(공유자원에 동시 접근을 제어하는 매커니즘) 등 필요 |
| 안정성 | 높음 (격리됨) | 낮음 (하나 죽으면 전체 영향) |
| 자원 사용 | 많음 | 적음 |
| 대표 사례 | Chrome 탭, Nginx worker | 웹 서버 요청 처리, Spring Boot |

### Spring에서 프로세스/스레드 사용

MVC 트래픽 관점에서 쓰레드 할당되어서 + 소켓바인딩

```java
┌─────────────────────────────────────────────────────────┐
│                 Spring Boot 애플리케이션                   │
│                    (단일 JVM 프로세스)                     │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Tomcat Thread Pool                 │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐            │   │
│  │  │Thread-1 │ │Thread-2 │ │Thread-3 │  ...       │   │
│  │  │ 요청 A   │ │ 요청 B   │ │ 요청 C   │            │   │
│  │  └────┬────┘ └────┬────┘ └────┬────┘            │   │
│  └───────┼───────────┼───────────┼─────────────────┘   │
│          ▼           ▼           ▼                     │
│  ┌─────────────────────────────────────────────────┐   │
│  │           HikariCP Connection Pool              │   │ # HTTP 호출할 때도 커넥션 풀이 있다.
│  │   [Conn1] [Conn2] [Conn3] [Conn4] [Conn5]       │   │
│  └─────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────┘
                          │
                          ▼
                    ┌──────────┐
                    │    DB    │
                    └──────────┘
```

- HikariCP

  *connection pooling을 제공하는 JDBC DataSource의 구현체*

- 커넥션 풀

  자바에서 DB에 직접 연결해서 처리하는 경우 JDBC Driver를 로드하고 커넥션 객체를 받아와야한다.

  매번 사용자가 요청을 할 때마다 드라이버를 로드하고 커넥션 객체를 생성해 연결하고 종료하기 때문에 비효율적이다. → 그래서 커넥션 풀을 사용한다.

**Pool의 최적을 찾는 방법**

**1. 단일 프로세스, 멀티 스레드**

```java
// Spring Boot는 하나의 JVM 프로세스에서 실행
// 각 HTTP 요청은 별도의 스레드에서 처리됨

@RestController
public class UserController {
    
    @GetMapping("/users/{id}")
    public User getUser(@PathVariable Long id) {
        // 이 메서드는 Tomcat 스레드 풀의 스레드에서 실행
        // 동시에 여러 요청이 오면 각각 다른 스레드가 처리
        return userService.findById(id);
    }
}
```

**2. 스레드와 트랜잭션/영속성 컨텍스트**

```java
@Service
public class UserService {
    
    @Transactional
    public void updateUser(Long id, String name) {
        // 트랜잭션과 EntityManager는 ThreadLocal에 바인딩됨 (JVM에서 힙사이즈를 정하는데, 개수에 맞춰서 해야 한다.)
        // ThreadLocal → 각 스레드마다 독립적인 영속성 컨텍스트(엔티티를 영구 저장하는 환경)를 가짐
        
        User user = userRepository.findById(id).orElseThrow();
        user.setName(name);
        // 같은 스레드 내에서는 동일한 영속성 컨텍스트 공유
    }
}
```

**3. 스레드 간 엔티티 공유 주의점**

```java
// ❌ 잘못된 예: 다른 스레드로 엔티티 전달
@Async
public void asyncProcess(User user) {
    // user 엔티티가 다른 스레드의 영속성 컨텍스트에서 생성됨
    // 여기서는 detached 상태 → LazyLoading 불가, 변경 감지 안됨
    user.getOrders().size(); // LazyInitializationException
}

// ✅ 올바른 예: ID만 전달하고 새로 조회
@Async
public void asyncProcess(Long userId) {
    // 이 스레드의 영속성 컨텍스트에서 새로 관리됨
    User user = userRepository.findById(userId).orElseThrow();

}
```

YML 설정 (많이 늘리면 늘릴수록 `TPS`가 떨어진다. → DB 병목이 발생될 것이다.)

```java
server:
  tomcat:
    threads:
      max: 200        # 최대 스레드 수, 디폴트가 200개
      min-spare: 10   # 최소 유지 스레드

spring:
  datasource:
    hikari:
      maximum-pool-size: 10    # DB 커넥션 풀 크기
      minimum-idle: 5
```

쓰레드가 많아질 수록 메모리 관점에서 ThreadLocal

히카리풀에서 슬로우쿼리 로그, 모니터링 할 방법은 뭘까? // JVM

Spring에서 트랜잭션 **ThreadLocal Bounding 찾기**

- Thread, 트랜잭션, 영속성 컨텍스트
- 그 다음이 EventListener 비동기, 이벤트 발생 / 비동기하고 쓰레드의 경계는?
