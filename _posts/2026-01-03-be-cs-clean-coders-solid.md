---
title: "클린 코더스"
date: 2026-01-03 16:50:00 +0900
categories: [cs, oop]
tags: [oop, solid]
---

## SOLID Foundation

### 목차

설계란

디자인 잘못되었을 때의 징후

코드가 점진적으로 썩어감

소프트웨어 공학의 결과물 : 바이너리 코드

소스코드 : document → 디자인

- 건물, 회로, 기계

  저렴한 설계 비용

  비싼 수정 비용

- 소프트웨어

  저렴한 구축 비용

  비싼 설계 비용

### Design Smells

1. Rigidity

   시스템의 의존성으로 인해 변경하기 어려워지는 것

   원인 :

   - 전체 리빌드를 유발하는 작은 변화
   - 많은 시간이 소요되는 테스트와 빌드

2. Fragility

   한 모듈의 수정이 다른 모듈에 영향을 미칠 때

   해결책 : 모듈간의 의존성을 제거

3. Immobility

## SRP

### Responsibility

클래스는 하나의 책임을 가져야 한다.

- 같은 책임

  save/findById → DB와 연결됨

  calculatePay, calcuateDeduction, calculateSalary

- 부류

  메서드의 client에 의해 결정 ⇒ 누가 해당 메소드의 변경을 유발하는 사용자인가 (누가 save를 유발하는가, 누가 find를 유발하는가)

**책임**

SW의 변경을 요청하는 특정 `Client`에 대해 함수/클래스가 갖는 것

→ 변경의 원인이 같다면 같은 책임이다.

**Two Values of SW**

Employee에 연결되어있는 클래스 : EmployeeUser1, EmployeeUser2, CalculationAPI, DatabaseAPI, StringAPI

- Primary Value of SW

  지속적으로 변화하는 요구사항을 수용

  대부분의 SW는 현재의 요구사항을 잘 만족하지만 변경하긴 어렵다.

- Collision

  동일한 모듈의 변경 → Merge 충돌

- FanOut

  Employee는 너무 많은 책임을 가지고 있다. → 변경에 민감하다.

**Collocation is Coupling**

기존의 책임에는 변경이 없어도 새로운 리포트가 추가되면서 Employee클래스가 변경된다.

→ 모든 액터들이 영향을 받게 된다.

### SRP

모듈은 **하나의** **변경 사유(Actor, 클래스를 사용하는 Client)**를 가져야 한다.

**시스템 설계**

Actor 파악에 주의해야 한다.

Actor들을 Serve하는 책임들을 식별하기

### Solution

가정 : EmployeeImpl에 Policy/Architect/Operations(3개의 Actor, 3개의 책임)

- 어떻게 책임을 분리할 것인가

  OOP에서 의존성을 다루는 전략 : 클래스를 인터페이스와 클래스로 분리

  1. Actor를 클래스에서 Decouple
     1. 모든 Actor들이 하나의 인터페이스에 coupled
     2. 하나의 클래스에 구현되어 구현도 coupled
  2. Extract Classes
     1. 3개의 클래스로 분리
  3. 3개의 클래스로 분리
     1. Actor들은 분리된 3개의 클래스에 의존
     2. 3개의 책임에 대한 구현은 분리

     문제점 : Employee의 개념이 3가지 조각으로 분리됨

  4. Facade

     구현이 어디에 있는지 찾기 쉽다.

     장점 : Actor들은 완전히 decoupled

     단점 : 어디에 구현되었는지 찾기 어렵다. 하나의 클래스에 구현되어 구현은 Coupled

### Case Study

### Faking it

WaterFall 순서로 할 것인가 X (액터 → 패키지 다이어그램 → 클래스 다이어그램 → 코드)

Step By Step으로 복잡한 것을 찾아낼 것인가

1. 테스트 작성 통과
2. 스코어 계산하는 함수

…

Unit Test가 확보된 후에 무차별적인 리팩토링 수행한다.

마지막에 다이어그램 그리기

## OCP

**Open and Closed**

> 확장에 대해 열려있고 변경에 대해선 닫혀있다.

확장 : 새로운 타입을 추가하면서 새로운 기능 추가

변경 : 확장이 될 때 상위 레벨이 영향을 받지 말아야 한다.

→ 소스코드 수정 없이 변경이 가능하다.

**Copy Example**

Copy Module을 컴파일 안하고 Low Level Details를 변경할 수 있다.

Abstract and Inversion (추상화와 역전)

**스프링** 프레임워크를 사용하면 `DI`를 편하게 해줌 → 테스트하기 편하고, 유지보수가 편하다

### **Example**

POS 예시 : 현금을 받는 프로그램은 잘 동작함, 신용카드를 받고자 할 때 확장을 해야 함

확장에 필요한 행위를 추상화하기(Interface로 주입하기)

**Is This Possible**

- OCP를 준수하면서 변경을 완벽하게 제거할 수 있냐

  이론적으로는 가능하다. 그러나 비실용적이다

  많은 분리가 발생하나, Crystal Ball(마법의 수정구_미래를 예측할 수 있는) 문제 발생

### Design

**BDUF(Big Design Up Front)**

고객의 요구사항을 반영할 수 있는, 예측하여 도메인 모델 생성

OCP가 가능하도록 도메인 모델에 추상화를 적용

→ 추상화는 유용하고 강력한 만큼 비용도 크다.

**Agile Design**

한번에 미리 필요한 것들을 다 생각하는게 아닌 점진적으로 예측하면서 Abstract 적용

리팩토링을 해서 변경을 쉽게 할 수 있도록 Abstraction을 추가 (OCP 준수할 수 있도록)

결론 : Iteration이 진행됨에 따라 OCP 적용율이 증가되어야 한다. (추상화가 증가된다.)

## LSP

상속 관계에서 자식은 예측 불가능한 행동을 하면 안된다.

**코드 예시**

```java
public class LSP {
	static P p = new P();
	
	static class T {
		public void doSomething() {
			System.out.println("T is called");
		}
	}
	
	static class S extends T {
		public void doSomething() {
			System.out.println("S is called");
		}
	}

	static class P {
		public void doSomething(T p) {
				p.doSomething();
		}
	}
	
	public static void main(String[] args) {
		T p = new T();
		S c = new S();
		
		p.doSomething(p);
		p.doSomething(c);
	}
}
```

**OCP와 LSP의 차이점**

- OCP

  Abstraction, polymorphism(Interitance)을 이용해서 구현

- LSP

  OCP를 받쳐주는 Ploymorphism에 관한 원칙 제공

  LSP가 위반되면 OCP도 위반됨

  LSP를 위반하면 Subtype이 추가될 때마다 클라이언트들이 수정되어야 함

  Instanceof/downcasting을 사용하는 것은 전형적인 LSP 위반의 징조

**The Representative Rule**

대리인은 자신이 대리하는 어떤 것들에 대한 관계까지 대리(공유)하지 않는다. (변호사가 A,B 모두 대리하지 않는 것처럼)

결론

DownCast, InstanceOf, 서브 클래스에서 Super클래스 호출

→ 상속관계를 끊고 Composition 관계로 고민해보기

## ISP

### **Interface Segregation Principle**

클라이언트 입장에서 인터페이스 분리하기

(사용하지 않는 것에 의존을 가지지 말기)

SRP를 잘 적용하기 위해 ISP를 적용할 수 있다.

**예제**

1. Switch 예제

   Switch는 Light에 의존적인가?

   Light뿐만 아니라 Fan, Motor등도 연관이 있나?

   → Switchable에 의존을 갖기 (구현체와는 관련이 없다.)

1. Fat Class 예제

   Job 클래스가 많은 일을 하고 있음

   → One Interface for a Sub System (Interface를 생성해서 Fat Class를 클라이언트로부터 격리시키기 == Fat Class에서 다수의 Interface를 구현)

   어떤 인퍼에티스에 변경이 발생하면 Job 클래스와 해당 인터페이스를 사용하는 서브 시스템만 Rebuild됨

## DIP

### Dependency Inversion Principle

상위 레벨의 정책은 하위 레벨의 상세에 의존하면 안된다.

Copy 로직(상위 레벨)은 IO 드라이버(하위 레벨의 상세)에 의존하면 안된다.

### OO의 핵심 == 의존성 관리

IoC를 통해 상위 레벨의 모듈을 하위 레벨의 모듈로 부터 보호하는 것

Inheritance, Encapsulation, Polymorphism은 핵심이 아니라 OO의 매커니즘

OO Design은 Dependecy Management

### Structured Design

- Top-Down 방법론 (소스코드의 의존성 방향 = 런타임 의존성의 방향)

  맨 하단의 모듈 하나의 변경이 발생하면 상단의 모듈이 영향을 받음 → 역전이 필요함

### Dependency Inversion

```java
A → B // A->B : CompileTime and Runtime 의존성
```

```java
A -> Interface <- B

// A -> Interface : CompileTime 의존성
// B -> Interface : CompileTime 의존성
// A -> B : Runtime 의존성
```

A는 Interface가 변경되지 않는 이상 B의 변경(구체적인 하위 레벨)으로부터 자유로움

### Plugins (Plugin 아키텍쳐)

의존성 역전이 SW 모듈간의 경계를 만드는 수단

- Boundary는 plugin을 만드는 방법과 같다.
- 어떤 의존성을 역전할지 고민

App Partition (Factory == Interface) - Main Partition (FactoryImpl)

FactoryImpl을 사용하는 모듈에 주입을 Main Partition이 해야 하는데, Spring이 한다.

### Architectural Implications [이해 잘 못함]

DIP가 아키텍쳐 측면에서 말하고자 하는 바

Uses - DB - Web 시스템을 분리하는 바운더리 간 의존성을 역전하기 위해 DIP가 사용된다.

## SOLID Case Study

### Requirement and use Cases

- 요구사항

  employee는 type에 따라 급여 주기가 다르다.

  employee type에 따라 disposition이 다르다.

1. SRP로 시작 (누가 어플리케이션의 액터인가? 그들의 관심사는?)

   Operations(시스템 운영), Union(어떻게 조합비 부과), Policy(얼마를 언제 수령)

   목적 : 모듈의 분리(각 모듈은 하나의 책임만 가진다.)

- SRP

  한 모듈의 하나의 Actor를 위한 책임을 갖도록 설계

  Actor를 찾고 이에 Module을 분리

- OCP

  Controller를 decoupling

- LSP

  문제점 : 새로운 지불 방법이 생길 때마나 Employee 클래스에 addXXX 메서드가 추가됨 (OCP 위반)

  해결 : interface에 PayType 추가

- ISP

- DIP

## 내가 이해한 SOLID

### 1. Single Responsibility Principle (단일 책임 원칙)

**"클래스는 하나의 책임만 가져야 한다"**

```java
// 나쁜 예: 여러 책임이 혼재
@Service
public class UserService {
    public User findUser(Long id) { ... }           // 유저 확인
    public void sendEmail(User user) { ... }        // 이메일 발송은 다른 책임
    public String generateReport(User user) { ... } // 리포트 생성도 다른 책임
}

// 좋은 예: 책임 분리
@Service
public class UserService {
    public User findUser(Long id) { ... }
    public User saveUser(User user) { ... }
}

@Service
public class EmailService {
    public void sendEmail(User user) { ... }
}

@Service
public class ReportService {
    public String generateReport(User user) { ... }
}
```

### 2. Open/Closed Principle (개방-폐쇄 원칙)

**"확장에는 열려있고, 수정에는 닫혀있어야 한다"**

```java
// 나쁜 예: 새로운 할인 타입 추가 시 코드 수정 필요
public class DiscountService {
    public double calculate(Order order) {
        if (order.getType().equals("VIP")) {
            return order.getPrice() * 0.2;
        } else if (order.getType().equals("NORMAL")) {
            return order.getPrice() * 0.1;
        }
        // 새 타입 추가 시 여기를 수정해야 함
        return 0;
    }
}

// 좋은 예: 인터페이스를 통한 확장
public interface DiscountPolicy {
    double calculateDiscount(Order order);
}

@Component
public class VipDiscountPolicy implements DiscountPolicy {
    @Override
    public double calculateDiscount(Order order) {
        return order.getPrice() * 0.2;
    }
}

@Component
public class NormalDiscountPolicy implements DiscountPolicy {
    @Override
    public double calculateDiscount(Order order) {
        return order.getPrice() * 0.1;
    }
}

// 새로운 정책 추가 시 기존 코드 수정 없이 클래스만 추가
@Component
public class PremiumDiscountPolicy implements DiscountPolicy {
    @Override
    public double calculateDiscount(Order order) {
        return order.getPrice() * 0.3;
    }
}
```

### 3. Liskov Substitution Principle (리스코프 치환 원칙)

**“상속 관계에서 예측 불가능한 행동은 삼가야 한다.”**

```java
// JpaRepository는 CrudRepository를 완벽히 대체 가능
public interface UserRepository extends JpaRepository<User, Long> {
    // JpaRepository의 모든 메서드가 CrudRepository 계약을 준수
}
```

### **4. Interface Segregation Principle (인터페이스 분리 원칙)**

**"클라이언트는 자신이 사용하지 않는 메서드에 의존하지 않아야 한다"**

- 예2)

```java
// Spring Data JPA는 인터페이스가 잘 분리되어 있음
CrudRepository      // 기본 CRUD 연산
PagingAndSortingRepository  // 페이징, 정렬 추가
JpaRepository       // JPA 특화 기능 추가

// 필요한 기능만 선택해서 상속 가능
public interface UserRepository extends CrudRepository<User, Long> {
    // 페이징이 필요 없으면 CrudRepository만 사용
}
```

- 예 1)

```java
// 나쁜 예: 거대한 인터페이스
public interface UserRepository {
    User findById(Long id);
    List<User> findAll();
    void save(User user);
    void delete(User user);
    void sendNotification(User user);  // Repository에 맞지 않는 메서드
    void generatePdf(User user);       // Repository에 맞지 않는 메서드
}

// 좋은 예: 역할별로 분리
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmail(String email);
}

public interface NotificationService {
    void sendNotification(User user);
}

public interface ReportGenerator {
    void generatePdf(User user);
}
```

### **5. Dependency Inversion Principle (의존성 역전 원칙)**

**"고수준 모듈은 저수준 모듈에 의존하면 안 되고, 둘 다 추상화에 의존해야 한다"**

예) @Autowired, 생성자 주입

```java
// 나쁜 예: 구체 클래스에 직접 의존
@Service
public class OrderService {
    private MySQLOrderRepository repository = new MySQLOrderRepository(); // 구체 클래스 의존
    
    public void createOrder(Order order) {
        repository.save(order);
    }
}

// 좋은 예: 인터페이스(추상화)에 의존 + DI 활용
public interface OrderRepository extends JpaRepository<Order, Long> {
}

@Service
public class OrderService {
    private final OrderRepository orderRepository; // 추상화에 의존
    
    // 생성자 주입 (Spring이 구현체를 주입)
    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }
    
    public void createOrder(Order order) {
        orderRepository.save(order);
    }
}

```
