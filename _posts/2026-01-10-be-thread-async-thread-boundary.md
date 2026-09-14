---
title: "@Async + 스레드 경계"
date: 2026-01-10 11:36:00 +0900
categories: [spring, async-thread]
tags: [spring, async, thread-pool]
---

호출된 메서드가 호출자와 다른 별도의 스레드에서 실행되도록 하여 명확한 스레드 경계를 형성합니다. 이는 AOP(관점 지향 프로그래밍) 프록시 패턴을 기반으로 구현됩니다.

### 동기 VS 비동기

동기(Synchronous)

- 호출 - 응답이 같은 스레드에서 순차 진행
- 제어 흐름이 직선적 → 이해/디버깅이 쉽다
- 입출력 대기 시간이 길면 리소스 낭비 → 스레드가 놀고 있음

비동기(Asynchronous)

- 호출 즉시 제어권 반환 → 작업은 다른 스레드에서 진행
- CPU, I/O 자원을 더 효율적으로 쓰지만 흐름이 분산되어 디버깅이 어렵다

**Spring 비동기의 핵심: AOP와 프록시(Proxy)**

스프링의 AOP는 프록시 객체로 메서드 호출을 가로챈다. @Async가 붙은 메서드를 호출하면 프록시가 이를 감지해 TaskExecutor에 작업을 등록한다.

### 비즈니스 요구사항

사용자 검증을 마치고 응답을 보낼 때 이메일 전송하기

예 1) 동기방식

스레드 없이 호출, 메일 전송이 끝날 때까지 해당 요청 스레드가 블로킹되어 응답이 늦어질 수 있다.

**코드**

```java
@Service
public class MailService {
    
    @Value("${spring.mail.username}")
    private String fromMail;
    
    private final JavaMailSender mailSender;
    
    public MailService(JavaMailSender mailSender) {
        this.mailSender = mailSender;
    }
    
    public void send(MailPayload payload) {
        SimpleMailMessage mail = new SimpleMailMessage();
        mail.setFrom(fromMail);
        mail.setTo(payload.email());
        mail.setSubject("인증 코드");
        mail.setText(payload.code());
        mailSender.send(mail);  // 메일 전송 완료될 때까지 여기서 대기
    }
}
```

예 2) new Thread()

스레드를 생성해서 메서드를 별도의 스레드에서 실행, 스레드 생성/소멸 비용이 크고 예외처리가 어렵다. 만약 스레드 수 제한이 없다면 시스템 리소스 고갈 위험이 크다. + 비즈니스 로직과 비동기 로직이 한 메서드에 포함되어있어 핵심 기능 작성에 집중하기 어렵다.

**코드**

```java
@Override
public void send(MailPayload payload) {
    new Thread(() -> {
        SimpleMailMessage mail = new SimpleMailMessage();
        mail.setFrom(fromMail);
        mail.setTo(payload.email());
        mail.setSubject("인증 코드");
        mail.setText(payload.code());
        mailSender.send(mail);
    }).start();
}
```

**비교**

| 방식 | 장점 | 단점 |
| --- | --- | --- |
| 동기 (스레드 X) | 단순함 | 메일 전송 동안 응답 지연 |
| `new Thread()` | 간단한 비동기 | 리소스 낭비, 관리 어려움 |
| `ExecutorService` | 스레드 재사용, 효율적 | 직접 관리 필요 |
| `@Async` | Spring이 관리, 깔끔 | 프록시 제약 있음 |

### @Async를 활용한 비동기 처리

**사용방법**

1. `@EnableAsync`  어노테이션으로 Spring에게 `@Async` 어노테이션을 활용할 것을 알린다.
2. 비동기로 동작해야 하는 메서드에 `@Async` 어노테이션을 붙인다.

```java
@Async
public void send(MailPayload payload) {
    SimpleMailMessage mail = new SimpleMailMessage();
    mail.setFrom(fromMail);
    mail.setTo(payload.email());
    mail.setSubject("인증 코드");
    mail.setText(payload.code());
    mailSender.send(mail);
}
```

### `@Async`의 원리

`@Async` 는 기본적으로 스프링 **AOP에 의해 프록시 패턴** 기반으로 동작한다.

1. `@Async` 어노테이션이 붙은 메서드가 호출되면 스프링은 해당 호출을 가로채서 비동기 실행을 처리하기 위한 프록시 객체를 생성한다.
2. 해당 메서드는 TaskExecutor에 의해 스레드풀에 작업으로 등록한다.
3. 해당 메서드는 호출자와 별도의 스레드에서 작업이 진행되며, 호출자 메서드는 블로킹되지 않고 즉시 리턴된다.

**두가지 조건**

@Async가 올바르게 동작하기 위해서 두 가지 조건이 필요하다.

1. public 메서드에서만 적용 가능하다.
2. 같은 클래스의 메서드를 호출할 수 없다. (Self-Invocation 불가)

### `@EnableAsync` 적용 방법

1. 구성 클래스에 직접 적용

@EnableAsync 어노테이션은 아래와 같이 구성 클래스에 붙일 수 있다. (별도의 TaskExecutor를 설정하지 않으면, SimpleAsyncTaskExecutor가 활용된다.)

```java
@EnableAsync
@SpringBootApplication
public class EmailApplication {

    public static void main(String[] args) {
        SpringApplication.run(TestApplication.class, args);
    }

}
```

1. AsyncConfigurer 인터페이스 활용

AsyncConfigurer 인터페이스를 구현하여 별도의 TaskExecutor를 설정해줄 수 있다.

```java
@EnableAsync
@Configuration
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(5);
        executor.setQueueCapacity(5);
        executor.setKeepAliveSeconds(30);
        executor.setThreadNamePrefix("async-executor-");
        executor.initialize();
        return executor;
    }
}
```

1. @Bean 활용

@Bean을 사용해 TaskExecutor를 직접 빈으로 등록해서 사용할 수 있다. 다수의 빈 등록이 가능하다.

```java
  @Bean(name = "queryExecutor")
  public ThreadPoolTaskExecutor queryExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(32);        // 4 vCPU 기준 “안정적”
    executor.setMaxPoolSize(64);         // burst 대응
    executor.setQueueCapacity(2000);     // 충분한 buffer
    executor.setThreadNamePrefix("QueryPool-");
    executor.setKeepAliveSeconds(30);
    executor.setWaitForTasksToCompleteOnShutdown(true);
    executor.setAwaitTerminationSeconds(30);
    executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
    executor.initialize();
    return executor;
  }
```

### **ThreadPoolTaskExecutor를 활용한 스레드 관리**

`@Async` 어노테이션을 사용할 때 별도의 TaskExecutor를 설정하지 않으면 `SimpleAsyncTaskExecutor`가 기본적으로 사용된다.

SimpleAsyncTaskExecutor는 스레드 풀을 사용하지 않고, 매 요청마다 새로운 스레드를 생성해 작업을 수행한다. 예) 1000명의 사용자가 동시에 인증 메일 전송 요청 시 1000개의 스레드를 생성하려 하고, 이는 리소스 부족 문제로 이어지기 쉽다.

⇒ ThreadPoolTaskExecutor와 같은 스레드 풀 기반의 TaskExecutor을 사용하도록 설정해야 한다.

**주요 옵션들**

- corePoolSize

  스레드 풀에 상항 살아있는 최소 스레드 수

  executor.setCorePoolSize(32);        // 4 vCPU 기준 “안정적”

- maxPoolSize

  스레드 풀이 확장할 수 있는 최대 스레드 수 : 스레드 풀 크기 + 초과 요청을 담는 큐의 크기가 넘는 요청이 들어온 경우 스레드 풀이 얼만큼 확장할 수 있는지

  executor.setMaxPoolSize(64);         // burst 대응

- queueCapacity

  스레드 풀에서 사용할 최대 큐의 크기

  executor.setQueueCapacity(2000);     // 충분한 buffer

- threadNamePrefix

  생성된 각 스레드의 이름 접두사

  executor.setThreadNamePrefix("QueryPool-");

- keepAliveSeconds

  스레드풀 내 스레드 개수가 corePoolSize 초과인 상태에서 대기 상태의 스레드가 종료되기까지의 시간

  executor.setKeepAliveSeconds(30);

SpringBoot를 사용할 경우 autoConfiguration으로 ThreadPoolTaskExecutor가 자동으로 등록되기 때문에 `application.yml`으로 executor의 옵션을 지정해주는 방법도 있다.

### **ThreadPoolTaskExecutor의 동작 방식**

![](/assets/img/2026-01-10/be-thread-async-thread-boundary-1.png)

1. 스레드 풀에 작업을 등록하면 스레드 풀에 corePoolSize만큼 스레드가 존재하는지 확인
2. 스레드에 작업 할당
   1. 스레드풀의 스레드 개수가 corePoolSize보다 적으면 스레드풀에 새로운 스레드를 생성하고 작업을 할당
   2. 스레드풀의 스레드 개수가 corePoolSize보다 크면 스레드풀의 대기 상태 스레드에게 작업을 할당
3. 스레드풀에 존재하는 모든 스레드가 작업 중이면 BlockingQueue에 작업을 넣어 대기
   1. BlockingQueue가 가득찬 경우 현재 스레드풀 스레드 수가 maxPoolSize를 넘지 않으면 새로운 스레드를 생성
   2. 스레드 풀의 스레드 수가 maxPoolSize에 도달한 상태에서 새로운 요청이 들어오면 더이상 스레드 생성이 불가하고, 큐에도 대기 불가 (TaskRejectedException)
4. 작업중인 스레드가 작업을 마치면 BlockingQueue에 대기중인 작업이 있는지 확인

   대기중인 작업이 있으면 해당 작업을 가져와 작업 수행

   대기중인 작업이 없으면 해당 스레드는 대기 상태로 진입. 만약 스레드풀의 스레드 개수가 corePoolSize보다 크면 keepAlive타임이 지나고 해당 스레드는 스레드풀에서 제거

### ThreadPoolTaskExecutor 예외처리

RejectedExecutionHandler를 통해 ThreadPoolTaskExecutor에서 스레드풀 내에서 더 이상 작업을 처리할 수 없을때의 예외 처리 전략을 설정할 수 있다.

RejectedExecutionHandler의 기본 전략은 AbortPolicy이다.

Spring에서 제공하는 전략과 그 특징들은 아래와 같다.

- AbortPolicy: TaskRejectedException을 발생시키며 종료한다.
- CallerRunsPolicy: 스레드풀을 호출한 스레드에서 처리한다.
- DiscardPolicy: 해당 요청들을 무시한다.
- DiscardOldestPolicy: 큐에 있는 가장 오래된 요청을 삭제하고 새로운 요청을 받아들인다. (queueCapacity가 0인 경우 StackOverFlowError가 발생한다.)

```java
executor
    .setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
```
