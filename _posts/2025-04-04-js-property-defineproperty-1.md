---
title: "Property + Object.defineProperty (1)"
date: 2025-04-04 19:44:00 +0900
categories: [javascript, object]
tags: [javascript, property]
---

## 1. Property

property는 해당 object의 특징 또는 속성이다. property는 보통 데이터 구조와 연관된 속성을 나타내는데, 2가지 종류가 있다.

- 인스턴스 프로퍼티
  - 클래스를 정의한 후 new 생성자로 만들어진 인스턴스에 접근가능한 프로퍼티
- 정적 프로퍼티
  - 클래스에 고정되어있는 프로퍼티
    - 인스턴스를 통한 접근 x, 클래스로 접근이 가능함.

프로퍼티는 이름 + 값이라는 쌍을 가지고 있다. 프로퍼티가 object를 가지고 있다는 것은 프로퍼티가 object reference를 가지고 있다는 의미이다. 

### 프로퍼티와 메서드

프로퍼티는 object를 위해서 데이터를 저장한다.

메서드는 object가 요청을 받을 수 있는 액션이며, 함수의 프로퍼티이다.

## 2. Property Attribute

자바스크립트 엔진은 property를 생성할 때 상태를 나타내는 `property attribute` 를 default값으로 자동 정의한다.

### 상태의 구성요소

- value
- writable
  - 할당 연산자로 속성의 값을 바꿀 수 있는지 === 갱신이 가능한지
- enumerable
  - 열거가 가능한지
- configurable
  - 속성 값을 변경할 수 있고, 객체에서 삭제가 가능한지 === 재정의가 가능한지

### property attribute란

property attribute는 `내부 슬롯`이다. 그리고 직접 접근이 불가능하다.

- [[Value]]
- [[Writable]]
- [[Eunumerable]]
- [[Configuable]]

**내부 슬롯이란**

내부 슬롯이란 자바스크립트 엔진의 구현 알고리즘을 설명하기 위해 ECMAScript 사양에서 사용하는 의사 프로퍼티/의사 메서드이다.

내부 슬롯/내부 메서드는 정의된대로 자바스크립트 엔진에서 실제 동작하지만, 개발자가 접근하거나 호출할 방법은 제공하지 않는다.

## 3. Property Attribute의 접근

위에서 언급했듯이 내부 슬롯에는 직접 접근이 불가능하다. 그러나 Property Attribute 내부 슬롯에는 간접적으로 접근할 수 있는 방법이 제공된다. 

### Object.getOwnPropertyDescriptor

```javascript
const object1 = {
  property1: 42,
};

const descriptor1 = Object.getOwnPropertyDescriptor(object1, "property1");

console.log(descriptor1);
// Expected output : { value: 42, writable: true, enumerable: true, configurable: true }

console.log(descriptor1.configurable);
// Expected output: true

console.log(descriptor1.value);
// Expected output: 42
```

**매개변수**

- 첫번째 매개변수 : 객체 참조
- 두번째 매개변수 : 프로퍼티 키

**반환값**

Property Descriptor 객체를 반환한다.

---
