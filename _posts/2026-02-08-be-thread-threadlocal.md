---
title: "ThreadLocal"
date: 2026-02-08 18:20:00 +0900
categories: [spring, async-thread]
tags: [java, threadlocal, concurrency]
---

### 문제상황

여러 스레드가 같은 인스턴스 필드에 접근하면 동시성 문제가 발생한다. (처음 스레드가 보관한 데이터가 사라지는 등) 이 때 스레드 로컬을 사용하여 해결할 수 있다.

- 동시성 문제

  같은 인스턴스 필드에 두개 이상의 스레드가 접근해서 값을 변경할 때, 저장한 값과 조회 값이 달라지는 현상. 지역변수가 아닌 객체가 하나밖에 없는 `싱글톤`이나 `static` 전역 변수에서 발생한다.

### ThreadLocal

해당 스레드만 접근할 수 있는 특별한 저장소를 말한다. ThreadLocal은 각 쓰레드마다 내부 저장소를 제공하고 관리하기에, 동시에 같은 인스턴스의 스레드 로컬 필드에 접근해도 문제가 없다.

![](/assets/img/2026-02-08/be-thread-threadlocal-1.png)

### Java ThreadLocal 동작 방식

`java.lang.ThreadLocal`

Java ThreadLocal은 HashMap 방식으로 동작한다. 각 쓰레드는 ThreadLocal의 내부 클래스인 ThreadLocalMap 객체를 참조 필드로 가지고 있기 때문에 해당 필드에 새로 생성한 맵을 저장해준다. (Thread마다 자신만의 저장소가 생긴다.)

```java
public class Thread implements Runnable {
    ...
    ThreadLocal.ThreadLocalMap threadLocals = null;
    ...
}
```

- ThreadLocalMap

  ![](/assets/img/2026-02-08/be-thread-threadlocal-2.png)
  ![](/assets/img/2026-02-08/be-thread-threadlocal-3.png)

  각 원소의 타입은 Entry, 즉 내부적으로 Entry 배열(해시 테이블)로 구현이 되어있다.

- Entry

  ThreadLocal의 내부 정적 클래스, value에는 동시성 문제를 해결하고자 하는 필드 값이 들어간다.

  생성자에서 해시코드 값을 계산하여 해당 자리에 초기화할 첫번째 값을 넣어주는데, 해시코드는 ThreadLocal 객체의 해시드 값을 활용해서 생성하고 있다.

- ThreadLocal 조회

  ![](/assets/img/2026-02-08/be-thread-threadlocal-4.png)

  값을 가져올 때 ThreadLocalMap을 꺼내고, ThreadLocal객체의 Entry 배열에서 값을 가져온다.

### 활용

```java
@Slf4j
public class ThreadLocalService {

    // private String nameStore;  동시성 문제가 발생하는 코드
    private final ThreadLocal<String> nameStore = new ThreadLocal<>();

    public String logic(String name) {
        log.info("저장 name={} -> nameStore={}", name, nameStore.get());
        nameStore.set(name);
        sleep(1000);
        log.info("조회 nameStore={}", nameStore.get());
        return nameStore.get();
    }
   ...
}
```

### 주의사항

ThreadLocal의 값은 사용 후 제거를 해야 한다.

그렇지 않으면 **WAS(톰캣)은 스레드풀**로 미리 스레드를 생성하는데, 이 때 문제가 발생한다.

![](/assets/img/2026-02-08/be-thread-threadlocal-5.png)

1. 사용자 A가 HTTP 요청을 보냄
   1. thread-A 스레드를 할당받아 ThreadLocal에 사용자 A 데이터 저장
2. WAS는 사용이 끝난 thread-A를 제거하고 다시 스레드 풀에 반납한다. 따라서 thread-A와 함께 ThreadLocal의 데이터도 남아있는다.
3. 사용자 B가 HTTP 요청을 보냄
   1. thread-A에 데이터를 조회하는데 A의 데이터가 저장되어있으므로 B의 요청인데, A값을 반환

   ⇒ remove() 메서드로 삭제 필요

### Spring ThreadLocal

**Spring이 ThreadLocal을 사용하는 주요 사례**

```java
// 1. Spring Security - 인증 정보
// Spring이 자동으로 설정하고 정리
SecurityContextHolder.getContext().getAuthentication();

// 2. Spring Transaction - 트랜잭션 정보
@Transactional
public void someMethod() {
    // Spring이 현재 스레드에 트랜잭션 정보를 ThreadLocal에 저장
}

// 3. Spring의 Request Scope
@Component
@Scope(value = "request", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class RequestScopedBean {
    // Spring이 요청별로 ThreadLocal 기반으로 관리
}
```

Spring이 관리한다는 의미:

- Spring이 자동으로 설정 (요청 시작 시)
- Spring이 자동으로 정리 (요청 종료 시)
- 개발자는 `remove()`를 직접 호출할 필요 없다.
