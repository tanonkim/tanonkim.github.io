---
title: "[책] 전략적 모놀리스와 마이크로서비스 - Ch.10 의도한대로 모놀리스 구축"
date: 2026-05-17 19:09:00 +0900
categories: [book, architecture]
tags: [monolith, ddd, hexagonal]
---

> 실제 서비스명, 도메인, 리소스 식별자 등은 보안을 위해 예시 값으로 바꿨습니다.
{: .prompt-info }

> 📖 전략적 모놀리스와 마이크로서비스 / 📅 2026-04-21

---

## 🎯 1단계: 진입 전 준비

**👁️ 이 챕터에서 다룰 내용을 예측하면?**

앞 챕터들(아키텍처 스타일, 메시지 버스)을 배웠으니 이번엔 실제로 모놀리스를 어떻게 "올바르게" 짓는지 구체적 방법이 나올 것 같다. DDD나 헥사고날을 모놀리스에 적용하는 이야기일 듯.

**🧠 내가 이미 알고 있는 것은?**

- 빅볼 오브 머드 = 무계획으로 자란 스파게티 코드
- **`바운디드 콘텍스트`** = DDD에서 경계를 명확히 나눈 단위
- W사는 현재 모놀리스 구조

---

## 📖 2단계: 독서 중 포착

**💡 핵심 개념**

- **모놀리스의 3가지 주된 목표** — ① 처음부터 올바르게 설정 ② 잘못된 모놀리스 바로잡기 ③ 모놀리스 → MSA 전환
- **빅볼 오브 머드** — 비즈니스 기능 모듈화 없이 자란 모놀리스. 새 기능 개발·수정이 갈수록 느려짐. **모듈화된 구조**를 채택하면 모놀리스도 수년간 변화를 지속할 수 있음
- **비즈니스 역량** — 비즈니스가 하는 일의 정의. 조직 구조가 바뀌어도 비즈니스 역량 자체는 변하지 않음. 아키텍처 분리의 기준이 됨
- **바운디드 콘텍스트** — 각 콘텍스트는 자체 DB를 소유해야 하며, 외부에서 직접 DB를 공유·접근하면 안 됨. 통합 수단은 REST / RPC / 메시징을 통한 공개 API만
- **전술적 패턴** (엄격히 적용 시 객체·시간적 결합 최소화)
  - 모듈 / 애그리거트 기반 엔티티 / 도메인 이벤트 / 도메인 서비스
- **DDD(2003) + 포트·어댑터(2004)** — 애자일 선언문과 같은 시대의 산물. 복잡한 비즈니스 로직을 다루는 핵심 도구

**🔴 컴포넌트 소스코드가 혼돈을 보이는 일반적 패턴**

| 문제 패턴 | 설명 |
| --- | --- |
| 기술 중심 모듈화 | `controller/`, `service/`, `repository/` 폴더로만 분리 (비즈니스 역량 기준 X) |
| 비아키텍처 | 명시적 설계 결정 없이 그냥 코드가 자람 |
| 단위 테스트 부재 | 변경 시 신뢰망 없음 |
| CRUD 중심 설계 | 도메인 로직 없이 save/find만 반복 |
| 깊게 얽힌 커플링 | 모듈 간 강력한 양방향 의존 |
| 단일 컴포넌트 다계층 분리 | 하나의 기능이 여러 레이어에 걸쳐 파편화 |

**🔖 인상 깊었던 문장**

> *"모든 시스템에 MSA가 필요한 건 아니다. 모놀리식 아키텍처를 올바르게 구현하려면 모든 바운디드 콘텍스트 간의 명확한 분리의 중요성을 이해하고 유지하는 게 핵심이다."*

> *"빅볼 오브 머드에서 모놀리스로 전환하려면 전략적 규율, 지능적인 전술, 인내심이 필요하다."*

---

## 🔄 3단계: 내 언어로 설명하기 (Feynman)

**👶 신입 개발자에게 이 챕터를 설명한다면?**

모놀리스가 나쁜 게 아니다. "무계획으로 자란 모놀리스"가 나쁜 거다. 마치 방을 청소 안 하고 10년 살면 쓰레기장이 되는 것처럼, 코드도 비즈니스 경계 없이 쌓이면 건드릴 수 없는 덩어리가 된다. 반대로 처음부터 "이 기능은 이 모듈" 경계를 명확히 하고, 모듈끼리 직접 DB를 공유하지 않고 API로만 소통하면 — 모놀리스 하나로도 몇 년치 비즈니스 변화를 버틸 수 있다.

**바운디드 콘텍스트 DB 분리를 한 문장으로?**

"내 방 물건은 내가 관리하고, 남의 방 물건이 필요하면 문을 두드려 부탁한다" — DB를 직접 공유하지 않고 API를 통해서만 소통하는 것.

---

## 🔗 4단계: 연결 & 적용

**🛠️ Spring Boot에서 커플링 해제 — 코드로 보기**

### 포트/어댑터 패턴 (헥사고날)

```java
// ❌ 계층형 — 도메인이 JPA에 직접 의존
@Service
public class OrderService {
    private final OrderJpaRepository jpaRepository; // 인프라에 묶임
    
    public void placeOrder(OrderCommand cmd) {
        jpaRepository.save(new OrderEntity(cmd)); // DB가 바뀌면 여기도 바뀜
    }
}
```

```java
// ✅ 포트/어댑터 — 도메인은 인터페이스(포트)만 안다

// [포트] 도메인이 선언 — "나는 이런 게 필요해"
public interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(OrderId id);
}

// [도메인 서비스] 포트에만 의존 — JPA인지 MongoDB인지 모름
@Service
public class OrderService {
    private final OrderRepository orderRepository; // 인터페이스만 의존
    
    public void placeOrder(OrderCommand cmd) {
        Order order = Order.create(cmd.getProductId(), cmd.getQuantity());
        orderRepository.save(order); // 누가 구현하든 상관없음
    }
}

// [어댑터] 포트를 실제로 구현 — JPA 로직은 여기만 안다
@Repository
public class JpaOrderRepository implements OrderRepository {
    private final OrderJpaRepository jpa;
    
    @Override
    public void save(Order order) {
        jpa.save(OrderJpaEntity.from(order)); // 도메인 ↔ 인프라 변환 책임
    }
}
```

> 💡 DB를 MySQL → MongoDB로 교체해도 `OrderService`는 **한 줄도 안 바뀐다.**

---

### CQRS — 읽기/쓰기 분리로 커플링 해제

```java
// [Command] 쓰기 — 상태 변경, 이벤트 발행
@Service
@Transactional
public class OrderCommandService {
    public void placeOrder(PlaceOrderCommand cmd) {
        Order order = Order.place(cmd.getProductId(), cmd.getQuantity());
        orderRepository.save(order);
        eventPublisher.publishEvent(new OrderPlacedEvent(order.getId())); // 이벤트로 분리
    }
}

// [Query] 읽기 — 조회 최적화, 쓰기 모델과 독립적으로 확장 가능
@Service
@Transactional(readOnly = true)
public class OrderQueryService {
    public List<OrderSummaryDto> getOrderList(Long userId) {
        return queryRepository.findOrderSummariesByUserId(userId); // 비정규화 뷰 직접 조회
    }
}
```

---

### 애그리거트 — 양방향 커플링 차단

```java
// ❌ 외부에서 내부 컬렉션 직접 수정 → 강결합
order.getLineItems().add(new LineItem(...));

// ✅ 애그리거트 루트를 통해서만 접근 → 불변성 보장
@Entity
public class Order { // Aggregate Root
    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderLineItem> lineItems = new ArrayList<>();
    
    // 외부는 이 메서드만 호출 — 내부 구조 캡슐화
    public void addItem(ProductId productId, int quantity, Money price) {
        lineItems.add(new OrderLineItem(productId, quantity, price));
    }
    
    // Spring Data가 자동으로 이벤트 발행
    @DomainEvents
    Collection<Object> domainEvents() {
        return List.of(new OrderItemAddedEvent(this.id));
    }
}
```

**🧱 기존 지식과 연결**

- **Ch.8 헥사고날 아키텍처** → 포트/어댑터로 실제 구현됨. DB 교체 시 어댑터만 바꾸면 됨
- **Ch.9 CQRS** → Command/Query 분리를 Spring Service 레이어에서 실현. 읽기 전용 `@Transactional(readOnly = true)`와 직결
- **Ch.9 이벤트 소싱** → `@DomainEvents`로 애그리거트에서 이벤트 발행, ApplicationEventPublisher로 흘려보냄
- **HikariCP `maximum-pool-size` 설정** → 모놀리스 전체가 스케일 아웃될 때 커넥션 수 계산이 중요했던 이유와 연결

**🔍 코드에서 점검해볼 포인트**

- `service/`, `repository/` 폴더 구조 → 기술 중심 모듈화인가, 비즈니스 역량 중심인가?
- 바운디드 콘텍스트 간에 JPA Entity를 직접 참조하는 부분이 있는가? (있다면 ID 참조로 교체 검토)
- Command/Query가 같은 Service에 섞여 있는가? → CQRS 분리 시작점

---

## 🎯 5단계: 복습 질문 (Active Recall)

> ⚠️ 지금 답 적지 말 것. 3일~1주일 후에 책 없이 답해보는 것이 진짜 복습이다.

1. **바운디드 콘텍스트가 자체 DB를 소유해야 한다는 원칙이 왜 중요한가? 모놀리스에서 DB를 공유하면 구체적으로 어떤 문제가 생기는가?**
2. **포트/어댑터 패턴에서 "포트"와 "어댑터"는 각각 누가 정의하고 누가 구현하는가? Spring Boot 코드로 설명할 수 있는가?**
3. **빅볼 오브 머드 패턴 6가지 중 W사 코드에 해당하는 것이 있는가? 있다면 어떻게 개선할 수 있는가?**

---

## ⭐ 종합 평가

| 항목 | 선택 |
| --- | --- |
| 이해도 | ⭐⭐⭐⭐ |
| 실무 적용도 | ⭐⭐⭐⭐⭐ |
| 재독 필요 | ⭐⭐⭐ |

**💬 한 줄 요약**

모놀리스의 적은 규모가 아니라 경계 없는 커플링이다. `바운디드 콘텍스트 + 포트/어댑터 + 애그리거트`로 경계를 지키면 모놀리스도 오래 간다.
