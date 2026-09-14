---
title: "Lombok 주의점"
date: 2026-01-22 00:46:00 +0900
categories: [spring, core]
tags: [lombok, java]
---

[blog-sample/lombok at master · cheese10yun/blog-sample](https://github.com/cheese10yun/blog-sample/tree/master/lombok#%EB%AC%B4%EB%B6%84%EB%B3%84%ED%95%9C-setter-%EB%82%A8%EC%9A%A9)

### 1. @AllArgsConstructor, @RequiredArgsConstructor

**버그 시나리오**

```java
@AllArgsConstructor
@RequiredArgsConstructor
@ToString
public class TestUser {
    public String id;
    public String password;
}

@AllArgsConstructor
@RequiredArgsConstructor
@ToString
public class TestUser {
    public String password; // 위치 변경
    public String id;
}
```

테스트 코드

```java
@Test
void test() {
    TestUser testUser = new TestUser("아이디", "패스워드");
    System.out.println("testUser = " + testUser);
}
```

필드의 위치 변경이 기존의 생성자에게 오류를 생기게 한다.

→ password에 id가 들어가고, id 필드에 password가 들어간다. 에러가 발생되지 않아 알기 어렵다.

**해결책**

- `@AllArgsConstructor` 와 `@RequiredArgsConstructor` 사용을 금지한다.
  - 혹시 `@Builder` 등을 이용하기 위함이라면, 접근 레벨을 낮춰놓자.
- 생성자를 직접 만든다.
- `@Builder` 를 적극 활용한다. (이는 **파라미터 순서가 아닌 이름**으로 값을 설정한다.)

### 2. 파라미터 없는 @EqualsAndHashCode

```java
@Setter
@AllArgsConstructor
@EqualsAndHashCode
public class TestUser {
    public String id;
    public String password;
}
```

TestUser에 `@EqualsAndHashCode`를 추가하면 lombok은 `equals()` 메서드와 `hashCode()` 메서드를 자동으로 구현한다.

### 4. @ToString의 순환참조

A 객체의 멤버로 B 객체가 있고, B 객체의 멤버로 A 객체가 있다면 둘은 무한으로 순환하여 참조하기에 `@ToString`의 사용을 조심해야 한다.

@ToString(exclude = “필드면”)으로 @ToString항목에서 제외하는 것도 방법이다.

### 5. @Setter

`@Setter` 는 모든 필드에 대한 Setter 를 만들어주는데, 편한 반면 변하지 않아야 할 필드에 대해서도 `Setter` 를 만들 수 있어 주의해야 한다.

특히 엔티티의 식별자와 같은 것은 거의 바뀔 일이 없으므로 주의해야 한다.

### 3. @Data

@Data에는 @ToString, @EqualsAndHashCode, @Getter, @Setter, @RequiredArgsConstructor 애노테이션이 합쳐져 있다.

### 4. @Value

모든 필드를 private final로 변경해주는 기능이 있지만, 이 또한 @EqualsAndHashCode, @AllArgsConstructor를 포함한다. ⇒ 조금 더 찾아보기

### 5. @Builder는 직접 만든 생성자에 사용

```java
@Setter
public class TestUser {
    public String id;
    public String password;

    @Builder
    public TestUser(String id, String password) {
        this.id = id;
        this.password = password;
    }
}
```

직접 만든 생성자에 `@Builder` 를 붙이면 더욱 안전하다.

`@Builder` 를 편하게 사용하기 위해 `@AllArgsConstructor`와 같이 사용하는데, `access = PRIVATE` 을 하면 웬만해서는 실수할 일이 없다.
