---
title: "Bean Scope"
date: 2026-02-22 00:51:00 +0900
categories: [spring, core]
tags: [spring, bean-scope]
---

### Spring Bean Scope

Spring에서 Bean Scope는 **빈이 생성되고 존재하는 범위**를 정의한다.

### **주요 Scope 종류**

- **Singleton**

  스프링 컨테이너에서 **단 하나의 인스턴스**만 생성하여 모든 요청에서 공유한다.

  ```java
  @Scope("singleton")
  @Component
  public class MyService { }
  ```

- **prototype**

  요청할 때마다 **새로운 인스턴스**를 생성한다. 컨테이너는 생성과 의존성 주입까지만 관여하고, 이후 관리하지 않는다.

  ```java
  @Scope("prototype")
  @Component
  public class MyService { }
  ```

- **request**

  HTTP요청 하나 당 하나의 빈 인스턴스가 생성되고, 요청이 끝나면 소멸된다.

- **session**

  HTTP 세션 하나당 하나의 빈 인스턴스가 유지된다.

- **application**

  ServletContext 하나당 하나의 빈 인스턴스가 생성된다.

대부분의 경우 SingleTon으로 충분하며, Spring의 기본전략이다.

prototype 빈을 Singleton 빈에 주입하면 prototype이 매번 새로 생성되지 않는 문제가 발생한다.

→ ObjectProvider나 @Lookup을 사용해야 한다.

```java
@Component
public class SingletonBean {
    private final ObjectProvider<PrototypeBean> provider;

    public void logic() {
        PrototypeBean bean = provider.getObject(); // 호출 시마다 새 인스턴스
    }
}
```
