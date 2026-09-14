---
title: "JPA Entity의 equals와 hashCode"
date: 2026-01-17 17:51:00 +0900
categories: [jpa, entity]
tags: [jpa, equals, hashcode]
---

### 동일성(메모리 주소) / 동등성(내용)

```java
// 동일성: 메모리 주소가 같은가? (== 연산자)
User a = new User("kim", "1234");
User b = a;  // b는 a와 같은 객체를 가리킴
System.out.println(a == b);  // true (같은 메모리 주소)

// 동등성: 내용이 같은가? (equals 메서드)
User c = new User("kim", "1234");
User d = new User("kim", "1234");
System.out.println(c == d);       // false (다른 메모리 주소)
System.out.println(c.equals(d));  // ??? (equals를 어떻게 구현했느냐에 따라 다름)
```

### 핵심 문제

JPA에서는 같은 DB row를 나타내는 객체라도 **영속성 컨텍스트(Session)가 다르면** 서로 다른 인스턴스로 인식된다. 예를 들어 Session1에서 조회한 객체 A와 Session2에서 조회한 객체 C는 같은 PK를 가지더라도 `A == C`는 `false`입니다.

예) 같은 트랜잭션

```java
// 같은 트랜잭션(같은 영속성 컨텍스트) 안에서
User a = userRepository.findById(1L);  // DB에서 조회 → 캐시에 저장
User b = userRepository.findById(1L);  // 캐시에서 가져옴 (DB 안 감)

System.out.println(a == b);  // true! 같은 객체입니다
```

예) 다른 트랜잭션

```java
// 첫 번째 트랜잭션
@Transactional
public User method1() {
    User a = userRepository.findById(1L);
    return a;  // 트랜잭션 끝 → a는 "준영속" 상태가 됨
}

// 두 번째 트랜잭션
@Transactional  
public void method2(User a) {
    User c = userRepository.findById(1L);  // 새 영속성 컨텍스트에서 조회
    
    System.out.println(a == c);       // false! 다른 인스턴스
    System.out.println(a.equals(c));  // ??? equals 구현에 따라 다름
}
```

a와 c는 DB의 같은 row를 나타내는데, 다른 객체로 인식된다. → 이게 왜 문제일까

1. Set 사용 시 문제

```java
Set<User> users = new HashSet<>();

User a = method1에서_가져온_User();  // id=1, 준영속
User c = method2에서_가져온_User();  // id=1, 영속

users.add(a);
users.add(c);

System.out.println(users.size());  // 2?? 1이어야 하는데!
```

같은 사용자인데, Set에는 2번 들어감 (Set은 중복을 허용하지 않는 컬렉션)

### **해결 방법 3가지**

**1. PK로만 구현**

- 장점: 간단하고 직관적
- 단점: 아직 영속화되지 않은(비영속) 객체는 PK가 없어서 비교 불가. Set에 비영속 객체를 넣고 나중에 영속화하면 문제 발생

  ```java
  @Entity
  public class User {
      @Id @GeneratedValue
      private Long id;
      private String username;
      private String password;
      
      @Override
      public boolean equals(Object o) {
          if (this == o) return true;
          if (!(o instanceof User)) return false;
          
          User user = (User) o;
          
          // id가 null이면 비교 불가
          if (id == null) return false;
          
          return id.equals(user.getId());
      }
      
      @Override
      public int hashCode() {
          return id != null ? id.hashCode() : 0;
      }
  }
  ```

**2. PK 제외하고 모든 속성으로 구현**

- 장점: 비영속 상태에서도 비교 가능
- 단점: 속성이 변경되면 동등성이 깨짐. 속성 조합이 유일성을 보장하지 않으면 다른 row와 같다고 판단될 수 있음

  ```java
  @Override
  public boolean equals(Object o) {
      if (this == o) return true;
      if (!(o instanceof User)) return false;
      
      User user = (User) o;
      
      return Objects.equals(username, user.username) 
          && Objects.equals(password, user.password);
  }

  @Override
  public int hashCode() {
      return Objects.hash(username, password);
  }
  ```

**3. 비즈니스 키 사용 (권장)**

- 변경 빈도가 낮고 유일성이 보장되는 속성(예: username, email 등)을 선택
- 위 두 방식의 문제를 모두 해결할 수 있음

  ```java
  @Entity
  public class User {
      @Id @GeneratedValue
      private Long id;
      
      @Column(unique = true)  // DB에서도 유일성 보장
      private String email;   // 비즈니스 키!
      
      private String username;
      private String password;
      
      @Override
      public boolean equals(Object o) {
          if (this == o) return true;
          if (!(o instanceof User)) return false;
          
          User user = (User) o;
          
          // email로만 비교!
          return Objects.equals(email, user.email);
      }
      
      @Override
      public int hashCode() {
          return email != null ? email.hashCode() : 0;
      }
  }
  ```

**Lombok의 `@EqualsAndHashCode`를 무심코 쓰면?** 모든 필드를 다 비교해서 위의 문제들이 발생할 수 있다. Entity에서는 직접 구현하는 게 안전하다.

### Lombok의 @EqualsAndHashCode

궁금한 점 : equals, hashCode가 사용되는 상황

→ 대부분 직접 호출하지 않아도 내부적으론 자동 호출된다.

**1. 컬렉션(Set, Map)에서 사용**

```java
Set<User> users = new HashSet<>();

User user1 = new User("kim@example.com", "김철수");
User user2 = new User("kim@example.com", "김철수");

users.add(user1);
users.add(user2);  // 내부적으로 equals() 호출해서 중복인지 확인

System.out.println(users.size());  
// equals 미구현: 2 (중복 허용됨 - 버그!)
// equals 구현: 1 (정상)
```

**2. 컬렉션의 contains(), remove()**

```java
List<User> users = new ArrayList<>();
User user1 = new User("kim@example.com", "김철수");
users.add(user1);

User user2 = new User("kim@example.com", "김철수");  // 같은 내용의 새 객체

// 내부적으로 equals() 호출
users.contains(user2);  // equals 미구현: false / 구현: true
users.remove(user2);    // equals 미구현: 삭제 안됨 / 구현: 삭제됨
```

**3. 객체 비교할 때 직접 호출**

```java
// 로그인 검증
User inputUser = new User(inputEmail, inputPassword);
User dbUser = userRepository.findByEmail(inputEmail);

if (inputUser.equals(dbUser)) {
    // 로그인 성공
}
// 변경 감지
User original = userRepository.findById(1L);
User modified = // ... 수정된 객체

if (!original.equals(modified)) {
    // 변경사항 있음 → 저장
    userRepository.save(modified);
}
```

**4. Stream의 distinct()**

```java
List<User> allUsers = // 중복 포함된 리스트

List<User> uniqueUsers = allUsers.stream()
    .distinct()  // 내부적으로 equals() 호출
    .collect(Collectors.toList());
```

**5. 테스트 코드에서 검증**

```java
@Test
void 사용자_생성_테스트() {
    User expected = new User("kim@example.com", "김철수");
    User actual = userService.createUser("kim@example.com", "김철수");
    
    // assertEquals 내부에서 equals() 호출
    assertEquals(expected, actual);
    
    // AssertJ
    assertThat(actual).isEqualTo(expected);
}
```

궁금한 점 2: hashCode()는 왜 같이 구현해야 하나?

규칙: `equals()`가 true면 `hashCode()`도 같아야 함

```java
// hashCode 미구현 시 문제
User user1 = new User("kim@example.com");
User user2 = new User("kim@example.com");

user1.equals(user2);      // true (equals는 구현함)
user1.hashCode();         // 12345 (기본값 - 메모리 주소 기반)
user2.hashCode();         // 67890 (다른 값!)

// HashMap이 찾는 방식:
// 1. hashCode()로 버킷(저장 위치) 찾기
// 2. 그 버킷 안에서 equals()로 비교

Set<User> set = new HashSet<>();
set.add(user1);           // hashCode 12345 버킷에 저장
set.contains(user2);      // hashCode 67890 버킷에서 찾음 → 없음! → false
```
