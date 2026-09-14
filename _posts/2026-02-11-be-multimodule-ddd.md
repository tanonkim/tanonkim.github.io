---
title: "멀티모듈의 장점 with DDD"
date: 2026-02-11 23:42:00 +0900
categories: [spring, multi-module]
tags: [multi-module, ddd]
---

### Bounded Context의 물리적 분리

DDD에서 가장 핵심적인 개념 중 하나인 Bounded Context를 Spring 멀티모듈로 자연스럽게 구현할 수 있다. 각 모듈이 하나의 Bounded Context가 되어, 도메인 간 경계가 코드 레벨에서 명확해진다.

```java
project-root/
├── order-domain/          # 주문 Bounded Context
├── payment-domain/        # 결제 Bounded Context
├── user-domain/           # 회원 Bounded Context
├── common/                # Shared Kernel
└── infrastructure/        # 인프라 계층
```

### 도메인 순수성 보장

도메인 모듈이 infrastructure에 대한 의존성(JPA, Spring)을 컴파일 타임에 차단할 수 있다. order-domain 모듈의 build.gradle에 JPA의존성을 아예 넣지 않으면, 개발자가 실수로 도메인 엔티티에 @Entity를 붙이는 것 자체가 불가능해진다. → DDD에서 강조하는 도메인 모델의 기술 독립성을 강제하는 수단이다.

### Ubiquitous Language의 모듈별 격리

같은 단어라도 Bounded Context마다 의미가 다를 수 있다. 예) User는 주문 컨텍스트에서 Orderer, 결제 컨텍스트에서 Payer일 수 있다. 멀티모듈로 분리하면 각 모듈 안에서 해당 컨텍스트의 유비쿼터스 언어에 맞는 클래스명과 용어를 자유롭게 사용할 수 있고, 이름 충돌도 발생하지 않는다.

### 의존 방향 제어(Layered / Hexagonal Architecture)

모듈 간 의존성을 build.gradle에서 명시적으로 선언하므로, DDD의 계층 구조나, 헥사고날 아키텍처의 의존 방향 규칙을 빌드 시스템이 강제한다

```java
// infrastructure 모듈
dependencies {
    implementation project(':order-domain')   // 인프라 → 도메인 (O)
}

// order-domain 모듈에는 infrastructure 의존성이 없음
// 도메인 → 인프라 (X) → 컴파일 에러 발생s
```

→ 도메인은 외부를 모른다는 원칙이 구조적 제약으로 보장된다.

### Anti-Corruption Layer 구현 용이

Bounded Context 간 통신 시 필요한 ACL(Anti-Corruption Layer)을 별도 모듈이나 모듈 내 adapter로 명확히 분리할 수 있다. 외부 컨텍스트의 모델이 내부 도메인을 오염시키는 것을 구조적으로 방지한다.

### Shared Kernel의 명시적 관리

여러 Bounded Context가 공유하는 값 객체나 도메인 이벤트를 `common` 또는 `shared-kernel` 모듈로 분리하면, DDD의 Shared Kernel 패턴이 코드에 그대로 드러난다. 어떤 개념이 공유되고 있는지가 모듈 구조만 봐도 파악되므로, 변경 시 영향 범위도 쉽게 추적할 수 있다.
